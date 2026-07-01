ocaml-wasm — Corresponding Source mirror
==================================================

This repository publishes the Corresponding Source for the WebAssembly
build of ocaml (license: LGPL-2.1-only) used in edgetools.io.

Contents
  build/      our build recipe: Dockerfile + helper scripts/config/patches.
              Rebuild with:  docker build build/
  upstream/   the exact upstream source archive(s) the build fetched,
              byte-identical and sha256-verified (see below).

Upstream sources:
  ocaml.tar.gz
    https://github.com/ocaml/ocaml/archive/refs/tags/5.4.1.tar.gz
    sha256 d4528517aaa1a44b8e2b1bc109a1ed0a5e0014f3ddc4feb8906b11a7e063e89a
