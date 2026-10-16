> **ESP-IDF port note:** This tree is *trimmed*. Only the 26
> `crypto_sign` arithmetic objects needed by libxeddsa are built
> (see `CMakeLists.txt`); the one-shot `crypto_sign*` API and the other
> ref10 subtrees were removed because libsodium provides them on ESP-IDF.
> The guide below documents where the original files come from, not the
> current layout.

# Extracting ref10 from SUPERCOP

To implement XEdDSA, access to low-level cryptographic functions is required, which are usually not exported by crypto-libraries.

ref10 by Daniel J. Bernstein is a solid implementation of these low-level functions. It comes as part of the [SUPERCOP](https://bench.cr.yp.to/supercop.html) benchmark and is not available as a standalone library.
