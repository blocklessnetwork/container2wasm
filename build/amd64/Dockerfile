ARG GRUB_VERSION=2.06
ARG BUSYBOX_VERSION=1.36.1
ARG LINUX_LOGLEVEL=7
FROM scratch AS assets
COPY . .

FROM ubuntu:22.04 AS gcc-x86-64-linux-gnu-base
RUN apt-get update && apt-get install -y gcc-x86-64-linux-gnu linux-libc-dev-amd64-cross git make

FROM golang:1.22-bullseye AS golang-base
RUN apt-get update -y
FROM golang:1.21-bullseye AS golang-1.21-base
RUN apt-get update -y 

FROM gcc-x86-64-linux-gnu-base AS linux-amd64-dev-common
RUN apt-get update && apt-get install -y gperf flex bison bc
RUN mkdir /work-buildlinux
WORKDIR /work-buildlinux
COPY --link --from=assets /linux.tar.gz /work-buildlinux/
WORKDIR /work-buildlinux/
RUN tar xzvf linux.tar.gz && mv linux-6.1 linux

FROM linux-amd64-dev-common AS linux-amd64-dev
RUN apt-get install -y libelf-dev
WORKDIR /work-buildlinux/linux
COPY --link --from=assets /assets/config/bochs/linux_x86_config ./.config
RUN make ARCH=x86 CROSS_COMPILE=x86_64-linux-gnu- -j$(nproc) all && \
    mkdir /out && \
    mv /work-buildlinux/linux/arch/x86/boot/bzImage /out/bzImage && \
    make clean

FROM linux-amd64-dev-common AS linux-amd64-config-dev
WORKDIR /work-buildlinux/linux
COPY --link --from=assets /assets/config/bochs/linux_x86_config ./.config
RUN make ARCH=x86 CROSS_COMPILE=x86_64-linux-gnu- olddefconfig

FROM scratch AS linux-amd64-config
COPY --link --from=linux-amd64-config-dev /work-buildlinux/linux/.config /


FROM gcc-x86-64-linux-gnu-base AS grub-amd64-dev
ARG LINUX_LOGLEVEL
ARG GRUB_VERSION
RUN apt-get update && apt-get install -y mkisofs xorriso wget bison flex python-is-python3 gettext
COPY --link --from=assets grub-${GRUB_VERSION}.tar.gz .
WORKDIR /work/
RUN tar zxvf grub-${GRUB_VERSION}.tar.gz
WORKDIR /work/grub-${GRUB_VERSION}
RUN ./configure --target=i386
RUN make -j$(nproc)
RUN make install
RUN mkdir -p /iso/boot/grub
COPY --link --from=linux-amd64-dev /out/bzImage /iso/boot/grub/
COPY --link --from=assets ./config/bochs/grub.cfg.template /
RUN cat /grub.cfg.template | LOGLEVEL=$LINUX_LOGLEVEL envsubst > /iso/boot/grub/grub.cfg
RUN mkdir /out && grub-mkrescue --directory ./grub-core -o /out/boot.iso /iso


FROM gcc-x86-64-linux-gnu-base AS busybox-amd64-dev
ARG BUSYBOX_VERSION
RUN apt-get update -y && apt-get install -y gcc bzip2
COPY --link --from=assets busybox-${BUSYBOX_VERSION}.tar.bz .
WORKDIR /work
RUN tar xzvf busybox-${BUSYBOX_VERSION}.tar.bz && mv busybox-${BUSYBOX_VERSION}.tar.bz busybox
WORKDIR /work/busybox
RUN make CROSS_COMPILE=x86_64-linux-gnu- LDFLAGS=--static defconfig && \
    make CROSS_COMPILE=x86_64-linux-gnu- LDFLAGS=--static -j$(nproc) && \
    mkdir -p /out/bin && mv busybox /out/bin/busybox && \
    make LDFLAGS=--static defconfig && \
    make LDFLAGS=--static -j$(nproc) && \
    for i in $(./busybox --list) ; do ln -s busybox /out/bin/$i ; done && \
    mkdir -p /out/usr/share/udhcpc/ && cp ./examples/udhcp/simple.script /out/usr/share/udhcpc/default.script

FROM golang-1.21-base AS runc-amd64-dev
WORKDIR /
COPY --link --from=assets /runc.tar.gz .
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    mkdir -p /go/src/github.com/opencontainers/ && \
    tar xzvf runc.tar.gz && mv runc-1.1.12  /go/src/github.com/opencontainers/runc &&\
    cd /go/src/github.com/opencontainers/runc&&\
    make static GOARCH=amd64 CC=gcc EXTRA_LDFLAGS='-s -w' BUILDTAGS="" EXTRA_LDFLAGS='-s -w' BUILDTAGS="" && \
    mkdir -p /out/ && mv runc /out/runc

FROM gcc-x86-64-linux-gnu-base AS tini-amd64-dev
# https://github.com/krallin/tini#building-tini
RUN apt-get update -y && apt-get install -y cmake
ENV CFLAGS="-DPR_SET_CHILD_SUBREAPER=36 -DPR_GET_CHILD_SUBREAPER=37"
WORKDIR /work
COPY --link --from=assets /tini.tar.gz /work/
WORKDIR /work/tini
ENV CC="x86_64-linux-gnu-gcc -static"
RUN cmake . && make && mkdir /out/ && mv tini /out/

FROM gcc-x86-64-linux-gnu-base AS vmtouch-amd64-dev
COPY --link --from=assets /vmtouch.tar.gz .
RUN tar xzvf vmtouch.tar.gz && mv vmtouch-1.3.1 vmtouch && \
    cd vmtouch && \
    CC="x86_64-linux-gnu-gcc -static" make && \
    mkdir /out && mv vmtouch /out/

FROM golang-base AS init-amd64-dev
COPY --link --from=assets /assets /work
WORKDIR /work
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    GOARCH=amd64 go build -ldflags "-s -w -extldflags '-static'" -tags "osusergo netgo static_build" -o /out/init ./cmd/init