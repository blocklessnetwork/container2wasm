ARG SOURCE_REPO=git@github.com:ktock/container2wasm.git
ARG SOURCE_REPO_VERSION=v0.6.5

ARG LINUX_LOGLEVEL=7
ARG INIT_DEBUG=true
ARG VM_MEMORY_SIZE_MB=128

ARG BINFMT_VERSION=qemu-v6.1.0

FROM scratch AS linux_assets
COPY . .

# FROM ubuntu:22.04 AS assets-base
# ARG SOURCE_REPO
# ARG SOURCE_REPO_VERSION
# RUN apt-get update && apt-get install -y git
#RUN git clone -b ${SOURCE_REPO_VERSION} ${SOURCE_REPO} /assets

FROM ubuntu:22.04 as ubuntu-base
RUN apt-get update -y


FROM ubuntu-base AS gcc-riscv64-linux-gnu-base
RUN apt-get install -y gcc-riscv64-linux-gnu libc-dev-riscv64-cross git make

FROM gcc-riscv64-linux-gnu-base AS linux-riscv64-dev-common
RUN  apt-get install -y gperf flex bison bc
WORKDIR /
RUN mkdir /work-buildlinux
COPY --link --from=linux_assets /linux.tar.gz /work-buildlinux/
WORKDIR /work-buildlinux/
RUN tar xzvf linux.tar.gz && mv linux-6.1 linux

FROM linux-riscv64-dev-common AS linux-riscv64-dev
WORKDIR /work-buildlinux/linux
COPY --link --from=linux_assets /assets/config/tinyemu/linux_rv64_config ./.config
RUN make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc) all && \
    mkdir /out && \
    mv /work-buildlinux/linux/arch/riscv/boot/Image /out/Image && \
    make clean

FROM linux-riscv64-dev-common AS linux-riscv64-config-dev
WORKDIR /work-buildlinux/linux
COPY --link --from=linux_assets /assets/config/tinyemu/linux_rv64_config ./.config
RUN make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- olddefconfig


FROM gcc-riscv64-linux-gnu-base AS busybox-riscv64-dev
RUN apt-get update -y && apt-get install -y gcc bzip2
WORKDIR /work
COPY --link --from=linux_assets /busybox-1.36.1.tar.bz /work/
RUN tar xjvf busybox-1.36.1.tar.bz && mv busybox-1.36.1 busybox
WORKDIR /work/busybox
RUN make CROSS_COMPILE=riscv64-linux-gnu- LDFLAGS=--static defconfig
RUN make CROSS_COMPILE=riscv64-linux-gnu- LDFLAGS=--static -j$(nproc)
RUN mkdir -p /out/bin && mv busybox /out/bin/busybox
RUN make LDFLAGS=--static defconfig
RUN make LDFLAGS=--static -j$(nproc)
RUN for i in $(./busybox --list) ; do ln -s busybox /out/bin/$i ; done
RUN mkdir -p /out/usr/share/udhcpc/ && cp ./examples/udhcp/simple.script /out/usr/share/udhcpc/default.script

FROM gcc-riscv64-linux-gnu-base AS tini-riscv64-dev
# https://github.com/krallin/tini#building-tini
RUN apt-get update -y && apt-get install -y cmake
ENV CFLAGS="-DPR_SET_CHILD_SUBREAPER=36 -DPR_GET_CHILD_SUBREAPER=37"
WORKDIR /work
COPY --link --from=linux_assets /tini.tar.gz /work/
RUN tar xzvf tini.tar.gz && mv tini-0.19.0 tini
WORKDIR /work/tini
ENV CC="riscv64-linux-gnu-gcc -static"
RUN cmake . && make && mkdir /out/ && mv tini /out/

FROM --platform=riscv64 tonistiigi/binfmt:${BINFMT_VERSION} AS binfmt
FROM scratch AS binfmt-riscv64
FROM scratch AS binfmt-base
COPY --link --from=binfmt /usr/bin/binfmt /usr/bin/
FROM binfmt-base AS binfmt-amd64
COPY --link --from=binfmt /usr/bin/qemu-x86_64 /usr/bin/
FROM binfmt-base AS binfmt-aarch64
COPY --link --from=binfmt /usr/bin/qemu-aarch64 /usr/bin/
FROM binfmt-base AS binfmt-arm
COPY --link --from=binfmt /usr/bin/qemu-arm /usr/bin/
FROM binfmt-base AS binfmt-i386
COPY --link --from=binfmt /usr/bin/qemu-i386 /usr/bin/
FROM binfmt-base AS binfmt-mips64
COPY --link --from=binfmt /usr/bin/qemu-mips64 /usr/bin/
FROM binfmt-base AS binfmt-ppc64le
COPY --link --from=binfmt /usr/bin/qemu-ppc64le /usr/bin/
FROM binfmt-base AS binfmt-s390
COPY --link --from=binfmt /usr/bin/qemu-s390 /usr/bin/
FROM binfmt-$TARGETARCH AS binfmt-dev

FROM golang:1.22-bullseye AS golang-base
RUN apt-get update -y
FROM golang:1.21-bullseye AS golang-1.21-base
RUN apt-get update -y 

FROM golang-1.21-base AS runc-riscv64-dev
RUN apt-get update -y && apt-get install -y gcc-riscv64-linux-gnu libc-dev-riscv64-cross git make gperf
WORKDIR /
COPY --link --from=linux_assets /runc.tar.gz .
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    mkdir -p /go/src/github.com/opencontainers/ && \
    tar xzvf runc.tar.gz && mv runc-1.1.12  /go/src/github.com/opencontainers/runc &&\
    cd /go/src/github.com/opencontainers/runc&&\
    make static GOARCH=riscv64 CC=riscv64-linux-gnu-gcc EXTRA_LDFLAGS='-s -w' BUILDTAGS="" && \
    mkdir -p /out/ && mv runc /out/runc

FROM gcc-riscv64-linux-gnu-base AS vmtouch-riscv64-dev
COPY --link --from=linux_assets /vmtouch.tar.gz .
RUN tar xzvf vmtouch.tar.gz && mv vmtouch-1.3.1 vmtouch && \
        cd vmtouch && \
        CC="riscv64-linux-gnu-gcc -static" make && \
        mkdir /out && mv vmtouch /out/

FROM golang-base AS init-riscv64-dev
COPY --link --from=linux_assets /assets/ /work
WORKDIR /work
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    GOARCH=riscv64 go build -ldflags "-s -w -extldflags '-static'" -tags "osusergo netgo static_build" -o /out/init ./cmd/init


FROM golang-base AS bundle-dev
ARG TARGETPLATFORM
ARG INIT_DEBUG
ARG OPTIMIZATION_MODE
ARG NO_VMTOUCH
ARG EXTERNAL_BUNDLE
COPY --link --from=linux_assets /assets /work
WORKDIR /work
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    go build -o /bin/create-spec ./cmd/create-spec
COPY --link --from=oci-image-src / /oci

FROM ubuntu-base AS rootfs-riscv64-dev
RUN apt-get install -y mkisofs
COPY --link --from=busybox-riscv64-dev /out/ /rootfs/
COPY --link --from=binfmt-dev / /rootfs/
COPY --link --from=runc-riscv64-dev /out/runc /rootfs/sbin/runc
#COPY --link --from=bundle-dev /out/ /rootfs/
COPY --link --from=init-riscv64-dev /out/init /rootfs/sbin/init
COPY --link --from=vmtouch-riscv64-dev /out/vmtouch /rootfs/bin/
COPY --link --from=tini-riscv64-dev /out/tini /rootfs/sbin/tini
RUN mkdir -p /rootfs/proc /rootfs/sys /rootfs/mnt /rootfs/run /rootfs/tmp /rootfs/dev /rootfs/var /rootfs/etc && mknod /rootfs/dev/null c 1 3 && chmod 666 /rootfs/dev/null
RUN mkdir /out/ && mkisofs -l -J -R -o /out/rootfs.bin /rootfs/

FROM scratch AS linux-riscv64
COPY --link --from=linux-riscv64-dev /out/Image /out/Image
COPY --link --from=linux-riscv64-config-dev /work-buildlinux/linux/.config /
COPY --link --from=rootfs-riscv64-dev /out/rootfs.bin //outrootfs.bin



