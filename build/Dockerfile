FROM emscripten/emsdk:5.0.7 AS builder

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

ARG VERSION=5.4.1
ENV VERSION=${VERSION}
ARG SOURCE_SHA256=d4528517aaa1a44b8e2b1bc109a1ed0a5e0014f3ddc4feb8906b11a7e063e89a
ENV DIRECTORY=ocaml-${VERSION}
ENV HOST_PREFIX=/opt/ocaml

WORKDIR /src

ADD https://github.com/ocaml/ocaml/archive/refs/tags/${VERSION}.tar.gz ocaml.tar.gz
RUN echo "${SOURCE_SHA256}  ocaml.tar.gz" | sha256sum -c - \
    && tar -xzf ocaml.tar.gz && rm ocaml.tar.gz

RUN cp -a ${DIRECTORY} ${DIRECTORY}-host
WORKDIR /src/${DIRECTORY}-host
RUN ./configure --prefix="$HOST_PREFIX" \
      --disable-native-compiler \
      --disable-ocamldoc \
      --disable-debugger \
      --without-zstd \
    && make -j"$(nproc)" \
    && make install

RUN set -eu; cd "$HOST_PREFIX"; \
    rm -f bin/ocamlrun bin/ocamlrund bin/ocamlruni bin/ocamlyacc \
          bin/ocamlobjinfo bin/ocamlobjinfo.byte bin/ocamlcmt \
          bin/ocamlprof bin/ocamlcp bin/ocamlmktop bin/ocamlmklib; \
    rm -rf lib/ocaml/compiler-libs lib/ocaml/expunge lib/ocaml/stublibs \
           lib/ocaml/libcamlrun_shared.so \
           lib/ocaml/unix lib/ocaml/str lib/ocaml/threads lib/ocaml/dynlink \
           lib/ocaml/runtime_events lib/ocaml/profiling \
           share/man man share/doc; \
    find lib/ocaml -maxdepth 1 \( -name '*.ml' -o -name '*.mli' \) -delete; \
    find lib/ocaml \( -name '*.cmt' -o -name '*.cmti' -o -name '*.a' \
           -o -name '*.cmxs' -o -name '*.so' \) -delete; \
    echo "trimmed install size:"; du -sh "$HOST_PREFIX"

WORKDIR /src/${DIRECTORY}
COPY js ./js

ENV PATH=${HOST_PREFIX}/bin:${PATH}

ENV CC_FOR_BUILD=gcc

ENV POST_JS=/src/${DIRECTORY}/js/post.js

RUN emconfigure ./configure \
      --host=wasm32-unknown-emscripten \
      --prefix="$HOST_PREFIX" \
      --disable-native-compiler \
      --disable-ocamldoc \
      --disable-debugger \
      --without-zstd \
      CC_FOR_BUILD=gcc \
      TARGET_LIBDIR="$HOST_PREFIX/lib/ocaml" \
      ac_cv_func_getrusage=no \
      ac_cv_func_times=no

ENV OCAML_LDFLAGS="\
    -lnodefs.js -lworkerfs.js \
    --preload-file ${HOST_PREFIX}@${HOST_PREFIX} \
    --post-js ${POST_JS} \
    -s WASM_BIGINT=1 \
    -s ALLOW_MEMORY_GROWTH=1 \
    -s INITIAL_MEMORY=134217728 \
    -s STACK_SIZE=5242880 \
    -s FORCE_FILESYSTEM=1 \
    -s EXIT_RUNTIME=0 \
    -s EXPORTED_RUNTIME_METHODS=callMain,FS,NODEFS,WORKERFS,ENV \
    -s INCOMING_MODULE_JS_API=noInitialRun,wasmBinary,locateFile,preRun,postRun,instantiateWasm,print,printErr,arguments \
    -s INVOKE_RUN=0 \
    -s MODULARIZE=1 \
    -s EXPORT_ES6=1 \
"

RUN python3 - <<'PY'
def patch(path, anchor, insert):
    s = open(path).read()
    assert anchor in s, f'{path}: anchor not found, revisit emscripten patch:\n{anchor!r}'
    open(path, 'w').write(s.replace(anchor, anchor + insert, 1))
    print('patched', path)

patch('runtime/unix.c',
      'static void* map_fixed(void* mem, uintnat size, int prot)\n{\n',
      '#ifdef __EMSCRIPTEN__\n'
      '  /* emscripten lacks MAP_FIXED remap; a reserve_only map is already\n'
      '     writable (PROT_NONE is ignored), so commit/decommit are no-ops. */\n'
      '  (void)size; (void)prot;\n'
      '  return mem;\n'
      '#endif\n')

patch('runtime/signals.c',
      'void * caml_init_signal_stack(void)\n{\n',
      '#ifdef __EMSCRIPTEN__\n'
      '  /* no working sigaltstack and no SIGSEGV on stack overflow under wasm;\n'
      '     skip the alt stack, return a non-NULL sentinel the caller frees. */\n'
      '  return malloc(1);\n'
      '#endif\n')

patch('runtime/signals.c',
      'void caml_free_signal_stack(void * signal_stack)\n{\n',
      '#ifdef __EMSCRIPTEN__\n'
      '  free(signal_stack);\n'
      '  return;\n'
      '#endif\n')
PY
RUN grep -q '#ifdef __EMSCRIPTEN__' runtime/unix.c \
    && [ "$(grep -c '#ifdef __EMSCRIPTEN__' runtime/signals.c)" = "2" ]

RUN emmake make EXE=.mjs SAK_BUILD='gcc -I runtime -o $(1) $(2)' runtime/build_config.h \
    && grep OCAML_STDLIB_DIR runtime/build_config.h \
    && grep -Eq '#define OCAML_STDLIB_DIR +"..*"' runtime/build_config.h

RUN emmake make -j"$(nproc)" \
      EXE=.mjs \
      SAK_BUILD='gcc -I runtime -o $(1) $(2)' \
      LDFLAGS="${OCAML_LDFLAGS}" \
      runtime/ocamlrun.mjs

RUN set -eu \
    && mkdir -p /dist \
    && ls -la runtime/ocamlrun.* \
    && cp runtime/ocamlrun.mjs  /dist/ocaml.mjs \
    && cp runtime/ocamlrun.wasm /dist/ocaml.wasm \
    && cp runtime/ocamlrun.data /dist/ocaml.data \
    && sed -i 's/ocamlrun\.wasm/ocaml.wasm/g; s/ocamlrun\.data/ocaml.data/g' /dist/ocaml.mjs \
    && grep -q 'ocaml\.wasm' /dist/ocaml.mjs \
    && ! grep -q 'ocamlrun\.wasm' /dist/ocaml.mjs \
    && grep -q 'ocaml\.data' /dist/ocaml.mjs \
    && ! grep -q 'ocamlrun\.data' /dist/ocaml.mjs \
    && [ "$(head -c4 /dist/ocaml.wasm | od -An -tx1 | tr -d ' \n')" = "0061736d" ] \
    && [ "$(stat -c%s /dist/ocaml.wasm)" -gt 262144 ]

FROM scratch AS exportor
COPY --from=builder /dist /dist
