ARG GRUB_VERSION=2.06
ARG BUSYBOX_VERSION=1.36.1
ARG LINUX_LOGLEVEL=7
ARG VM_MEMORY_SIZE_MB=128
ARG GOPROXY=https://goproxy.cn
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
COPY --link --from=assets /grub-${GRUB_VERSION}.tar.gz /work/
WORKDIR /work/
RUN tar zxvf grub-${GRUB_VERSION}.tar.gz
WORKDIR /work/grub-${GRUB_VERSION}
RUN ./configure --target=i386
RUN make -j$(nproc)
RUN make install
RUN mkdir -p /iso/boot/grub
COPY --link --from=linux-amd64-dev /out/bzImage /iso/boot/grub/
COPY --link --from=assets /assets/config/bochs/grub.cfg.template /
RUN cat /grub.cfg.template | LOGLEVEL=$LINUX_LOGLEVEL envsubst > /iso/boot/grub/grub.cfg
RUN mkdir /out && grub-mkrescue --directory ./grub-core -o /out/boot.iso /iso


FROM gcc-x86-64-linux-gnu-base AS busybox-amd64-dev
ARG BUSYBOX_VERSION
RUN apt-get update -y && apt-get install -y gcc bzip2
COPY --link --from=assets busybox-${BUSYBOX_VERSION}.tar.bz /work/busybox-${BUSYBOX_VERSION}.tar.bz
WORKDIR /work
RUN tar xjvf busybox-${BUSYBOX_VERSION}.tar.bz && mv busybox-${BUSYBOX_VERSION} busybox
WORKDIR /work/busybox
RUN make CROSS_COMPILE=x86_64-linux-gnu- LDFLAGS=--static defconfig && \
    make CROSS_COMPILE=x86_64-linux-gnu- LDFLAGS=--static -j$(nproc) && \
    mkdir -p /out/bin && mv busybox /out/bin/busybox && \
    make LDFLAGS=--static defconfig && \
    make LDFLAGS=--static -j$(nproc) && \
    for i in $(./busybox --list) ; do ln -s busybox /out/bin/$i ; done && \
    mkdir -p /out/usr/share/udhcpc/ && cp ./examples/udhcp/simple.script /out/usr/share/udhcpc/default.script

FROM golang-1.21-base AS runc-amd64-dev
ARG GOPROXY
RUN if test "${GOPROXY}" = "true" ; then go env -w GOPROXY=${GOPROXY},direct ; fi
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
RUN tar xzvf tini.tar.gz && mv tini-0.19.0 tini
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
ARG GOPROXY
COPY --link --from=assets /assets /work
RUN if test "${GOPROXY}" = "true" ; then  go env -w GOPROXY=${GOPROXY},direct ; fi
WORKDIR /work
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    GOARCH=amd64 go build -ldflags "-s -w -extldflags '-static'" -tags "osusergo netgo static_build" -o /out/init ./cmd/init


FROM ubuntu AS bios-amd64-dev
RUN apt-get update && apt-get install -y build-essential git
COPY --link --from=assets Bochs Bochs
WORKDIR /Bochs/bochs
RUN CC="x86_64-linux-gnu-gcc" ./configure --enable-x86-64 --with-nogui
RUN make -j$(nproc) bios/BIOS-bochs-latest bios/VGABIOS-lgpl-latest
RUN mkdir /out/ && mv bios/BIOS-bochs-latest bios/VGABIOS-lgpl-latest /out/

FROM ubuntu:22.04 AS bochs-config-dev
ARG VM_MEMORY_SIZE_MB
RUN apt-get update && apt-get install -y gettext-base && mkdir /out
COPY --link --from=assets /assets/config/bochs/bochsrc.template /
RUN cat /bochsrc.template | MEMORY_SIZE=$VM_MEMORY_SIZE_MB envsubst > /out/bochsrc

FROM scratch 
COPY --link --from=busybox-amd64-dev /out/ /rootfs/
COPY --link --from=runc-amd64-dev /out/runc /rootfs/sbin/runc
COPY --link --from=init-amd64-dev /out/init /rootfs/sbin/init
COPY --link --from=vmtouch-amd64-dev /out/vmtouch /rootfs/bin/
COPY --link --from=tini-amd64-dev /out/tini /rootfs/sbin/tini
COPY --link --from=bios-amd64-dev /out/ /pack/
COPY --link --from=grub-amd64-dev /out/boot.iso /pack/
COPY --link --from=bochs-config-dev /out/bochsrc /pack/
