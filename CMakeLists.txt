cmake_minimum_required(VERSION 3.20)

# libxeddsa crypto backend:
#   - libsodium (espressif__libsodium) provides SHA-512, X25519, detached
#     Ed25519, key conversions and all randomness (ESP32 hardware RNG).
#   - the vendored ref10 tree below contributes ONLY the raw Ed25519
#     field/curve/scalar arithmetic that libsodium does not export (see
#     ref10/CMakeLists.txt for the exact file set and why).

# 1. Define the source files for the xeddsa library
set(xeddsa_sources
    src/ed25519.c
    src/pub_conversion.c
    src/pub_derivation.c
    src/seed_priv_utils.c
    src/utils.c
    src/version.c
    src/x25519.c
    src/xeddsa.c
)

# 2. Register this directory as an ESP-IDF component
idf_component_register(
    SRCS "${xeddsa_sources}"
    INCLUDE_DIRS "include" "ref10/include"
    PRIV_REQUIRES "espressif__libsodium"
)

target_compile_options(${COMPONENT_LIB} PRIVATE "-Wno-error=unused-function" "-Wno-error=parentheses")

# 3. Build the trimmed ref10 crypto_sign arithmetic (static library
#    'crypto_sign_arith'). The 8 other ref10 subtrees that used to be built
#    here are dead weight on ESP-IDF (libsodium provides them) and were
#    removed - see ref10/CMakeLists.txt.
add_subdirectory(ref10)

# 4. Private include directories needed by the component's sources.
#    (${COMPONENT_LIB} is a variable created by idf_component_register)
target_include_directories(${COMPONENT_LIB} PRIVATE
    "ref10/crypto_sign"
)

# 5. Add compile definitions needed for the build.
target_compile_definitions(${COMPONENT_LIB} PRIVATE
    BUILD
    SODIUM_DLL_EXPORT
)

# 6. Link the component library against the ref10 arithmetic library.
#    All other ref10 static libraries were removed together with the
#    subtrees that produced them.
target_link_libraries(${COMPONENT_LIB} PRIVATE
    crypto_sign_arith
)
