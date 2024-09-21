ARG RUSTUP_DIST_SERVER=https://rsproxy.cn
ARG RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup
ARG GOPROXY=https://goproxy.cn

FROM scratch AS assets
COPY . .

FROM rust:1.74.1-buster AS rust-common
ARG RUSTUP_DIST_SERVER
ARG RUSTUP_UPDATE_ROOT
COPY <<-EOF /etc/sources.list
deb http://mirrors.ustc.edu.cn/debian/ buster main
deb http://mirrors.ustc.edu.cn/debian/debian-security buster/updates main
deb http://mirrors.ustc.edu.cn/debian/debian buster-updates main
EOF
RUN apt-get update -y
WORKDIR /work
COPY --link --from=assets wasi-vfs  wasi-vfs
COPY <<-EOF /work/wasi-vfs/.cargo/config.toml
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"

replace-with = 'tuna'
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
EOF
RUN cd wasi-vfs &&\
    cargo build --package wasi-vfs-cli --release && \
    mkdir -p /tools/wasi-vfs/  && \
    mv target/release/wasi-vfs /tools/wasi-vfs/ && \
    cargo clean

WORKDIR /work
COPY --link --from=assets wizer wizer
COPY <<-EOF /work/wizer/.cargo/config.toml
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"

replace-with = 'tuna'
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
EOF
WORKDIR /work/wizer/
RUN cargo build --bin wizer --all-features --release && \
    RUSTFLAGS="-C link-arg=-s" cargo build --bin wizer --all-features && \
    mkdir -p /tools/wizer/ && \
    mv include target/release/wizer /tools/wizer/ && \
    mv target/debug/wizer /tools/wizer/wizer-debug && \
    cargo clean


FROM golang:1.22-bullseye AS golang-base
ARG GOPROXY
COPY --link --from=assets /assets/ /work/
RUN if test "${GOPROXY}" = "true" ; then go env -w GOPROXY=${GOPROXY},direct ; fi
WORKDIR /work
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    go build -o /bin/create-spec ./cmd/create-spec

FROM scratch as output
COPY --link --from=rust-common /tools/wasi-vfs /tools/wasi-vfs
COPY --link --from=rust-common /tools/wizer /tools/wizer
COPY --link --from=golang-base /bin/create-spec /tools/create-spec

FROM output

