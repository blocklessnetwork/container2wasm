ARG WASI_SDK_VERSION=19
ARG WASI_SDK_VERSION_FULL=${WASI_SDK_VERSION}.0
ARG WASI_VFS_VERSION=0.3.0
ARG RUSTUP_DIST_SERVER=https://rsproxy.cn
ARG RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup

FROM scratch AS assets
COPY . .

FROM rust:1.74.1-buster AS rust-common
COPY <<-EOF /etc/sources.list
deb http://mirrors.ustc.edu.cn/debian/ buster main
deb http://mirrors.ustc.edu.cn/debian/debian-security buster/updates main
deb http://mirrors.ustc.edu.cn/debian/debian buster-updates main
EOF
RUN apt-get update -y

FROM rust-common AS bochs-dev-common
ARG WASI_VFS_VERSION
ARG WASI_SDK_VERSION
ARG WASI_SDK_VERSION_FULL
ARG RUSTUP_DIST_SERVER
ARG RUSTUP_UPDATE_ROOT
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


