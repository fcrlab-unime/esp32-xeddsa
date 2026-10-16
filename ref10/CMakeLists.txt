cmake_minimum_required(VERSION 3.20)

# ---------------------------------------------------------------------------
# ref10 (Daniel J. Bernstein) - TRIMMED for the ESP-IDF port
#
# libxeddsa only needs the raw field / curve / scalar arithmetic of
# crypto_sign/ed25519/ref10 (fe_*, ge_*, sc_*). Everything else that used to
# be built here is provided by libsodium:
#
#   - crypto_core / crypto_stream / crypto_rng / crypto_hash /
#     crypto_hashblocks / crypto_verify / crypto_scalarmult  -> libsodium
#   - fastrandombytes / kernelrandombytes (random source)    -> libsodium
#     (the kernelrandombytes/urandom.c fallback opened /dev/urandom, which
#     does not exist on ESP32; it is now not built at all)
#
# The one-shot ref10 API (crypto_sign_ed25519_ref10[_open/_keypair],
# keypair.c/open.c/sign.c) is never called by libxeddsa either, so those
# objects are dropped as well. The 26 files below are the exact transitive
# symbol closure (verified by compiling every object and following the
# undefined-symbol references) of what src/ uses:
#
#   fe_frombytes fe_tobytes fe_add fe_sub fe_mul fe_invert fe_1
#   ge_scalarmult_base ge_p3_tobytes
#   sc_reduce sc_muladd
#
# The closure is fully self-contained: it defines every symbol it needs and
# references nothing outside the tree (no libc, no other ref10 module).
#
# Do NOT add kernelrandombytes/randombytes back without a working ESP32
# random source - all randomness in this project must come from libsodium.
# ---------------------------------------------------------------------------

# Shared integer types + the IMPORT/EXPORT (INTERFACE) macros live in
# ref10/include/ and are required by every file in crypto_sign/.
include_directories(include)

set(crypto_sign_arith_sources
    crypto_sign/fe_0.c
    crypto_sign/fe_1.c
    crypto_sign/fe_add.c
    crypto_sign/fe_cmov.c
    crypto_sign/fe_copy.c
    crypto_sign/fe_frombytes.c
    crypto_sign/fe_invert.c
    crypto_sign/fe_isnegative.c
    crypto_sign/fe_mul.c
    crypto_sign/fe_neg.c
    crypto_sign/fe_sq.c
    crypto_sign/fe_sq2.c
    crypto_sign/fe_sub.c
    crypto_sign/fe_tobytes.c
    crypto_sign/ge_madd.c
    crypto_sign/ge_p1p1_to_p2.c
    crypto_sign/ge_p1p1_to_p3.c
    crypto_sign/ge_p2_dbl.c
    crypto_sign/ge_p3_0.c
    crypto_sign/ge_p3_dbl.c
    crypto_sign/ge_p3_to_p2.c
    crypto_sign/ge_p3_tobytes.c
    crypto_sign/ge_precomp_0.c
    crypto_sign/ge_scalarmult_base.c
    crypto_sign/sc_muladd.c
    crypto_sign/sc_reduce.c
)

add_library(crypto_sign_arith_objects OBJECT ${crypto_sign_arith_sources})
set_property(TARGET crypto_sign_arith_objects PROPERTY POSITION_INDEPENDENT_CODE TRUE)
target_compile_definitions(crypto_sign_arith_objects PRIVATE BUILD)
target_include_directories(crypto_sign_arith_objects PUBLIC crypto_sign)
add_library(crypto_sign_arith STATIC $<TARGET_OBJECTS:crypto_sign_arith_objects>)
