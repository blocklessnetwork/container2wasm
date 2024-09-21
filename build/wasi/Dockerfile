ARG WASI_SDK_VERSION=19
ARG WASI_SDK_VERSION_FULL=${WASI_SDK_VERSION}.0
ARG WASI_VFS_VERSION=0.3.0
ARG BINARYEN_VERSION=114
ARG RUSTUP_DIST_SERVER=https://rsproxy.cn
ARG RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup
ARG INIT_DEBUG=true

FROM scratch AS assets
COPY . .

FROM rust:1.74.1-buster AS rust-common
COPY <<-EOF /etc/sources.list
deb http://mirrors.ustc.edu.cn/debian/ buster main
deb http://mirrors.ustc.edu.cn/debian/debian-security buster/updates main
deb http://mirrors.ustc.edu.cn/debian/debian buster-updates main
EOF
RUN apt-get update -y

FROM rust-common AS tinyemu-dev-common
ARG WASI_VFS_VERSION
ARG WASI_SDK_VERSION
ARG WASI_SDK_VERSION_FULL
ARG RUSTUP_DIST_SERVER
ARG RUSTUP_UPDATE_ROOT
ARG BINARYEN_VERSION
RUN apt-get install -y make git xz-utils

WORKDIR /wasi

COPY --link --from=assets wasi-sdk.tar.gz wasi-sdk.tar.gz
RUN tar xvf wasi-sdk.tar.gz && rm wasi-sdk.tar.gz
ENV WASI_SDK_PATH=/wasi/wasi-sdk-${WASI_SDK_VERSION_FULL}

WORKDIR /work/
COPY --link --from=assets wasi-vfs wasi-vfs
COPY <<-EOF /work/wasi-vfs/.cargo/config.toml
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"

replace-with = 'tuna'
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
EOF
RUN cd wasi-vfs &&\
    cargo build --target wasm32-unknown-unknown && \
    mkdir -p /tools/wasi-vfs/ && \
    mv target/wasm32-unknown-unknown/debug/libwasi_vfs.a /tools/wasi-vfs/ && \
    cargo clean

WORKDIR /work/
COPY --link --from=assets tinyemu tinyemu
COPY --link --from=assets wizer.h /tools/wizer/include/wizer.h
WORKDIR /work/tinyemu
RUN make -j $(nproc) -f Makefile \
    CONFIG_FS_NET= CONFIG_SDL= CONFIG_INT128= CONFIG_X86EMU= CONFIG_SLIRP= \
    CC="${WASI_SDK_PATH}/bin/clang --sysroot=${WASI_SDK_PATH}/share/wasi-sysroot -D_WASI_EMULATED_SIGNAL -DWASI -I/tools/wizer/include/" \
    EMU_LIBS="/tools/wasi-vfs/libwasi_vfs.a -lrt" \
    EMU_OBJS="virtio.o pci.o fs.o cutils.o iomem.o simplefb.o json.o machine.o temu.o wasi.o riscv_machine.o softfp.o riscv_cpu32.o riscv_cpu64.o fs_disk.o"

WORKDIR /work/
COPY --link --from=assets Bochs Bochs
COPY --link --from=assets binaryen.tar.gz binaryen.tar.gz
WORKDIR /work/Bochs/bochs/wasi_extra/jmp
RUN mkdir /jmp && cp jmp.h /jmp/
RUN ${WASI_SDK_PATH}/bin/clang --sysroot=${WASI_SDK_PATH}/share/wasi-sysroot -O2 --target=wasm32-unknown-wasi -c jmp.c -I . -o jmp.o
RUN ${WASI_SDK_PATH}/bin/clang --sysroot=${WASI_SDK_PATH}/share/wasi-sysroot -O2 --target=wasm32-unknown-wasi -Wl,--export=wasm_setjmp -c jmp.S -o jmp_wrapper.o
RUN ${WASI_SDK_PATH}/bin/wasm-ld jmp.o jmp_wrapper.o --export=wasm_setjmp --export=wasm_longjmp --export=handle_jmp --no-entry -r -o /jmp/jmp

WORKDIR /work/Bochs/bochs/wasi_extra/vfs
RUN mkdir /vfs
RUN ${WASI_SDK_PATH}/bin/clang --sysroot=${WASI_SDK_PATH}/share/wasi-sysroot -O2 --target=wasm32-unknown-wasi -c vfs.c -I . -o /vfs/vfs.o
WORKDIR /work/Bochs/bochs
ARG INIT_DEBUG
RUN LOGGING_FLAG=--disable-logging && \
    if test "${INIT_DEBUG}" = "true" ; then LOGGING_FLAG=--enable-logging ; fi && \
    CC="${WASI_SDK_PATH}/bin/clang" CXX="${WASI_SDK_PATH}/bin/clang++" RANLIB="${WASI_SDK_PATH}/bin/ranlib" \
    CFLAGS="--sysroot=${WASI_SDK_PATH}/share/wasi-sysroot -D_WASI_EMULATED_SIGNAL -DWASI -D__GNU__ -O2 -I/jmp/ -I/tools/wizer/include/" \
    CXXFLAGS="${CFLAGS}" \
    ./configure --host wasm32-unknown-wasi --enable-x86-64 --with-nogui --enable-usb --enable-usb-ehci \
    --disable-large-ramfile --disable-show-ips --disable-stats ${LOGGING_FLAG} \
    --enable-repeat-speedups --enable-fast-function-calls --disable-trace-linking --enable-handlers-chaining --enable-avx # TODO: --enable-trace-linking causes "out of bounds memory access"
RUN make -j$(nproc) bochs EMU_DEPS="/tools/wasi-vfs/libwasi_vfs.a /jmp/jmp /vfs/vfs.o -lrt"
RUN mkdir -p /binaryen
RUN tar -C /binaryen -zxvf /work/binaryen.tar.gz
RUN /binaryen/binaryen-version_${BINARYEN_VERSION}/bin/wasm-opt bochs --asyncify -O2 -o bochs.async --pass-arg=asyncify-ignore-imports
RUN mv bochs.async bochs

COPY --link --from=joing/bls-c2w-tools /tools /tools
RUN cp bochs bochs-org && /tools/wizer/wizer --allow-wasi --wasm-bulk-memory=true -r _start=wizer.resume --mapdir /pack::/pack -o bochs_wizer bochs-org


FROM scratch 
COPY --link --from=tinyemu-dev-common /tools/wasi-vfs /out/wasi-vfs
COPY --link --from=tinyemu-dev-common /work/tinyemu/temu /out/temu
COPY --link --from=tinyemu-dev-common /work/Bochs/bochs/bochs_wizer /out/bochs_wizer
COPY --link --from=tinyemu-dev-common /work/Bochs/bochs/bochs /out/bochs



