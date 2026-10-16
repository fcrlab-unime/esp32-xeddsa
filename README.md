# libxeddsa for ESP-IDF

A native port of [**libxeddsa**](https://github.com/Syndace/libxeddsa) (the only
known C implementation of the **XEdDSA** signature scheme) to the
Espressif **ESP-IDF**, packaged as a drop-in ESP-IDF component. It lets an
ESP32 compute the XEd25519 signatures that the X3DH protocol requires
directly on the silicon — no external gateway, no desktop-only code paths.

XEdDSA (the X25519–Ed25519 "hybrid" signature) is the signature used by X3DH
to bind a device's signed pre-key to its identity key. Because it is not part
of MbedTLS or wolfSSL, and because libxeddsa was only ever built for
desktop operating systems, no ready-to-use ESP32 build existed. This repository
provides one.

---

## License & provenance

This project is licensed under the **MIT License** — see [`LICENSE`](LICENSE).

The tree combines two upstream bodies of code plus the porting work, and the
license preserves each one's provenance:

| Part of the tree | Upstream | License |
|---|---|---|
| `src/*.c`, `include/xeddsa.h` | [Syndace/libxeddsa](https://github.com/Syndace/libxeddsa) v2.0.1, **byte-identical** | MIT © 2025 Tim Henkes (Syndace) — notice preserved as MIT requires |
| `ref10/crypto_sign/*` | Daniel J. Bernstein's **ref10** (extracted from SUPERCOP) | **Public domain** — no copyright claim |
| Build system, trim, `ESP_PLATFORM` handling, docs | This port | MIT © 2025 Antonino Mastronardo |

> **Is MIT compatible with the original Syndace license?** Yes. Syndace's
> libxeddsa is MIT, which is a permissive license: it only asks that you keep
> the copyright + permission notice in the files you redistribute. This repo
> does exactly that — `src/` and `include/` are copied verbatim with the
> original notice retained in [`LICENSE`](LICENSE). The ref10 arithmetic is
> public domain and carries no restriction. Distributing the combined tree
> under MIT is therefore fully compliant.

---

## What the component provides

libxeddsa exposes a small, well-defined C API (declared in
[`include/xeddsa.h`](include/xeddsa.h)):

- **Public-key derivation / conversion**
  - `priv_to_curve25519_pub()` — X25519 public key from a Curve25519/Ed25519 secret
  - `priv_to_ed25519_pub()` — Ed25519 public key from the same secret
  - `seed_to_ed25519_pub()` — Ed25519 public key from a 32-byte seed
  - `curve25519_pub_to_ed25519_pub()` / `ed25519_pub_to_curve25519_pub()` — the birational map between the two encodings
- **Key adjustment**
  - `priv_force_sign()` — negates the scalar (constant-time) so the derived Ed25519 public key has the requested sign bit
- **Signatures**
  - `ed25519_priv_sign()` — XEdDSA sign with an externally supplied 64-byte nonce
  - `ed25519_seed_sign()` — standard Ed25519 sign from a seed
  - `ed25519_verify()` — verify
- **Diffie–Hellman**
  - `x25519()` — X25519 shared-secret computation
- **Initialization**
  - `xeddsa_init()` — idempotent setup (wraps `sodium_init()`)

---

## Why a port was needed

The upstream library was "built for and tested exclusively on Linux, macOS,
Windows and WebAssembly (Emscripten)". Three things made it unusable on a
bare-metal/RTOS target like the ESP32 as-is:

1. **Desktop build system.** Its CMake produced static *and dynamic* shared
   libraries and depended on an autotools-generated `sodium/version.h`. None of
   that maps onto the ESP-IDF component model.
2. **A `/dev/urandom` landmine.** The vendored ref10 tree contained a
   `kernelrandombytes/` fallback that opens `/dev/urandom` for entropy. That
   device node does not exist on the ESP32; if that code path were compiled in
   and reached at runtime it would fail (or hang) on a system with no Linux
   kernel.
3. **Desktop import/export macros.** The `cross_platform.h` header selected
   linking markers from `__APPLE__`/`__linux__`/… and would otherwise `#error`
   on an unrecognized platform.

The port resolves all three without altering a single byte of the original
`src/` or `include/` sources.

---

## How the port is structured

The component is a thin hybrid. Two backends supply the mathematics, and the
component only wires them together:

- **`espressif__libsodium`** provides everything that libsodium already
  exports: SHA-512, X25519 (`crypto_scalarmult_curve25519`), detached Ed25519
  sign/verify, the key-derivation/conversion helpers, **and all randomness**
  (the ESP32 hardware RNG, via `sodium_init()` / `randombytes_*`).
- **The trimmed `ref10/crypto_sign` tree** supplies the *only* low-level
  Ed25519 arithmetic that libsodium does **not** export: the raw field
  (`fe_*`), curve (`ge_*`) and scalar (`sc_*`) operations, used directly by the
  libxeddsa `src/` files to implement `priv_to_ed25519_pub()`,
  `ed25519_priv_sign()`, `priv_force_sign()`, etc.

```
libxeddsa  (this component)
├── src/            ← unchanged libxeddsa C sources (MIT, Syndace)
├── include/        ← unchanged public header (MIT, Syndace)
└── ref10/          ← Bernstein ref10, TRIMMED to 26 crypto_sign objects (public domain)
        └── builds → static lib `crypto_sign_arith`

dependencies:  espressif__libsodium  (registry)  → SHA-512, X25519, Ed25519, RNG
```

---

## The porting changes, in detail

### 1. Build-system rewrite (`CMakeLists.txt`, `ref10/CMakeLists.txt`)

The desktop CMake was replaced with the ESP-IDF component model:

- `idf_component_register(...)` declares this directory as a first-class
  component with `INCLUDE_DIRS "include" "ref10/include"`.
- `PRIV_REQUIRES "espressif__libsodium"` injects a private dependency on the
  **registry** build of libsodium, so the crypto primitives and the hardware
  RNG resolve inside the ESP-IDF ecosystem rather than through autotools.
- The dynamic/shared-library targets that made no sense on a bare-metal/RTOS
  firmware were removed entirely (there is no `dlopen` on the target).
- Two private compile definitions are added:
  - `BUILD` — makes `cross_platform.h` treat the symbols as `EXPORT` rather
    than `IMPORT` (the "building the library, not importing a DLL" case).
  - `SODIUM_DLL_EXPORT` — the marker libsodium uses for its public symbols.

### 2. The ref10 trim

The upstream ref10 tree ships nine subtrees. On the ESP32, eight of them plus
the one-shot `crypto_sign` API are dead weight because **libsodium already
provides them**. The port keeps exactly the transitive closure of the
field/curve/scalar arithmetic the libxeddsa `src/` files actually call.

The 26 objects that are built (into the static library `crypto_sign_arith`):

| Group | Objects |
|---|---|
| `fe_*` (field) | `fe_0 fe_1 fe_add fe_cmov fe_copy fe_frombytes fe_invert fe_isnegative fe_mul fe_neg fe_sq fe_sq2 fe_sub fe_tobytes` |
| `ge_*` (curve) | `ge_madd ge_p1p1_to_p2 ge_p1p1_to_p3 ge_p2_dbl ge_p3_0 ge_p3_dbl ge_p3_to_p2 ge_p3_tobytes ge_precomp_0 ge_scalarmult_base` |
| `sc_*` (scalar) | `sc_muladd sc_reduce` |

Everything else in ref10 was deleted:

- **Whole subtrees removed** (libsodium provides these):
  `crypto_core`, `crypto_hash`, `crypto_hashblocks`, `crypto_rng`,
  `crypto_scalarmult`, `crypto_stream`, `crypto_verify`, `fastrandombytes`,
  **`kernelrandombytes`** (see next item).
- **One-shot API removed** (never called by libxeddsa): `keypair.c`, `open.c`,
  `sign.c` and their headers (`api.h`, `base2.h`, `d.h`, `d2.h`), plus the
  curve helpers outside the closure (`ge_add`, `ge_sub`, `ge_msub`,
  `ge_frombytes`, `ge_tobytes`, `ge_double_scalarmult`, `ge_p2_0`,
  `ge_p3_to_cached`, `fe_isnonzero`, `fe_pow22523`).

> The closure is not a guess. It was determined by compiling every object and
> following the undefined-symbol references until the set was self-contained:
> the 26 objects define every symbol they need and reference nothing outside
> the tree. Adding any one of the removed files back would re-introduce the
> duplicate symbols that libsodium already provides.

### 3. Removing the `/dev/urandom`

`ref10/kernelrandombytes/` (and the `fastrandombytes/` wrapper) were deleted in
their entirety. The consequence:

- The code that would `open("/dev/urandom", ...)` is **not compiled at all**,
  so it cannot fail on a target with no `/dev`.
- **All** randomness now flows through libsodium, which on the ESP-IDF is
  backed by the chip's **hardware RNG**. There is exactly one entropy source —
  libsodium — and it is the one the rest of the firmware already trusts.

This is a hard rule of this component: do **not** re-add
`kernelrandombytes`/`randombytes` without a working ESP32 entropy source. All
randomness must come from libsodium.

### 4. `cross_platform.h` — teaching the preprocessor about ESP-IDF

One line was added to the POSIX branch so the header recognizes the target
(and stops falling through to its `#error "Unsupported operating system"`):

```c
// upstream:
#elif defined(__APPLE__) || defined(__linux__) || defined(__unix__) || defined(_POSIX_VERSION)
// this port:
#elif defined(__APPLE__) || defined(__linux__) || defined(__unix__) || defined(_POSIX_VERSION) || defined(ESP_PLATFORM)
```

This is what lets the same header compile cleanly under the Xtensa GCC toolchain:
on `ESP_PLATFORM` (as on POSIX/UNIX) no `__declspec(import/export)` marking is
required, so `IMPORT`/`EXPORT` resolve to empty and `INTERFACE` to the
`BUILD`-selected value.

### 5. The two compiler flags

The top-level `CMakeLists.txt` carries two private compile options:

```cmake
target_compile_options(${COMPONENT_LIB} PRIVATE "-Wno-error=unused-function" "-Wno-error=parentheses")
```

The ESP-IDF build turns on `-Werror`, so any diagnostic in third-party code
would abort the build. These two flags demote exactly two harmless, known
upstream diagnostics back to warnings. Both are deliberately written as
**`-Wno-error=<name>`**, not `-Wno=<name>`:

- **`-Wno-error=<name>` keeps the warning visible** in the build log — a
  reviewer can still see it — but stops it from failing the build. `-Wno=<name>`
  would erase it entirely, which would hide a real regression later.
- Both are **`PRIVATE`**, so the relaxation applies only to *this* component's
  objects and does not leak into the rest of the firmware.

The two diagnostics, with their exact source and the reason each is benign:

| Flag | Source | What the compiler complains about | Why it is safe |
|---|---|---|---|
| `-Wno-error=parentheses` | `src/utils.c:45` `scalar_out[31] = scalar_in[31] & 127 \| 64;` | "parentheses around `a & b \| c` are confusing" | The expression is Bernstein's clamping step. `&` binds tighter than `\|`, so it already evaluates as the intended `(scalar_in[31] & 127) \| 64`. The logic is correct; the warning is purely stylistic. |
| `-Wno-error=unused-function` | `src/utils.c` `sc_clamp`, `sc_cmov`, `sc_neg` | "static function defined but not used" | An upstream design quirk: `utils.c` is `#include`d *as text* by `ed25519.c`, `pub_derivation.c` and `seed_priv_utils.c` **and** is also compiled as its own translation unit. In the standalone TU those three `static` helpers are genuinely unused (they are used only in the TUs that include the file), so the compiler flags them. The code is correct; only the standalone TU is "unused". |

The flags were validated with an A/B compile: removing them makes the build fail
on exactly the lines above; with them, the component compiles cleanly under the
full `-Werror` toolchain.

---

## What changed vs. upstream, at a glance

| Status | Paths |
|---|---|
| **Added** (port) | top-level `CMakeLists.txt`, this `README.md`, `LICENSE` |
| **Modified** (port) | `ref10/CMakeLists.txt` (rewritten for the trimmed set), `ref10/README.md` (port note), `ref10/include/cross_platform.h` (`+ESP_PLATFORM`) |
| **Deleted** (trim) | 8 ref10 subtrees + `kernelrandombytes`/`fastrandombytes`, the one-shot `crypto_sign` API, and the curve helpers outside the 26-object closure |
| **Unchanged** | all of `src/`, all of `include/`, and every kept `ref10/crypto_sign` file (byte-identical to upstream) |

---

## Requirements

- **ESP-IDF ≥ 5.3** (tested against 5.5.x with the Xtensa GCC toolchain).
- **`espressif__libsodium`** from the ESP Component Registry — the component
  declares it as a private dependency, but your *project* must make it
  available. In your project's `idf_component.yml`:

  ```yaml
  dependencies:
    espressif/libsodium: "^1.0.20~3"
  ```

- The ESP32 hardware RNG (available by default) as the entropy source.

---

## Using the component

### Option A — local component

Copy this repository's contents into your project's `components/` directory
so that the folder is named `libxeddsa`:

```
your_project/
└── main/
└── components/
    └── libxeddsa/        ← this repository's contents
```

Then, in the component that uses it, add `libsodium` and `libxeddsa` to `REQUIRES` (or `PRIV_REQUIRES`).


### Option B — via the ESP Component Manager (optional)

To pull this as a remote git dependency you would add an `idf_component.yml`
manifest to the repository and reference it from your project, e.g.:

```yaml
dependencies:
  libxeddsa:
    git: "https://github.com/fcrlab-unime/esp32-xeddsa.git"
    path: "."
```

(Option A is self-contained and needs no manifest, which is why the repository
is shipped without one.)

---

## Verification

The port's correctness does not rest on trust. Two independent checks were
performed:

1. **Byte-identity.** Every kept file in `src/` and `include/` is
   byte-identical to upstream libxeddsa v2.0.1, and every kept
   `ref10/crypto_sign` file is byte-identical to Bernstein's ref10. The only
   differences from upstream are the build files, the trim, and the one-line
   `cross_platform.h` addition listed above.
2. **Known-answer test (KAT).** A deterministic on-target KAT was run against
   golden vectors generated by an unmodified upstream build on x86, where the
   upstream build was itself anchored byte-for-byte to the **RFC 8032**
   TEST 1 / TEST 2 Ed25519 vectors before any vector was emitted. All 64
   checks (5 key sets × 12 operations, plus the 4 RFC 8032 anchors, plus
   negative controls) passed, confirming the trimmed ref10 arithmetic is
   bit-identical to upstream.

---

## Repository layout

```
esp32-xeddsa/
├── CMakeLists.txt          # ESP-IDF component registration + build glue
├── include/
│   └── xeddsa.h            # public API (unchanged upstream, MIT/Syndace)
├── src/                    # libxeddsa C sources (unchanged upstream, MIT/Syndace)
│   ├── ed25519.c
│   ├── pub_conversion.c
│   ├── pub_derivation.c
│   ├── seed_priv_utils.c
│   ├── utils.c
│   ├── version.c
│   ├── x25519.c
│   └── xeddsa.c
├── ref10/                  # Bernstein ref10, trimmed (public domain)
│   ├── CMakeLists.txt      # builds the 26-object crypto_sign_arith library
│   ├── include/            # cross_platform.h (+ESP_PLATFORM), crypto_* int types
│   └── crypto_sign/        # the 26 fe_*/ge_*/sc_* objects + their headers
├── README.md
└── LICENSE                 # MIT (Syndace core + this port)
```

---

## Credits

- **Tim Henkes (Syndace)** — original [libxeddsa](https://github.com/Syndace/libxeddsa).
- **Daniel J. Bernstein** — the [ref10](https://bench.cr.yp.to/supercop.html)
  implementation of Curve25519/Ed25519 (public domain, via SUPERCOP).
- **The libsodium authors** (via the Espressif `espressif__libsodium`
  registry component) — SHA-512, X25519, Ed25519 and the hardware-RNG-backed
  randomness.
- **Antonino Mastronardo** — ESP-IDF port: build-system rewrite, ref10 trim,
  entropy-source fix, and documentation.
