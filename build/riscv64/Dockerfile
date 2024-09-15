ARG SOURCE_REPO=git@github.com:ktock/container2wasm.git
ARG SOURCE_REPO_VERSION=v0.6.5

FROM scratch AS linux_assets
COPY . .

# FROM ubuntu:22.04 AS assets-base
# ARG SOURCE_REPO
# ARG SOURCE_REPO_VERSION
# RUN apt-get update && apt-get install -y git
#RUN git clone -b ${SOURCE_REPO_VERSION} ${SOURCE_REPO} /assets


FROM ubuntu:22.04 AS gcc-riscv64-linux-gnu-base
RUN apt-get update && apt-get install -y gcc-riscv64-linux-gnu libc-dev-riscv64-cross git make

FROM gcc-riscv64-linux-gnu-base AS linux-riscv64-dev-common
RUN apt-get update && apt-get install -y gperf flex bison bc
WORKDIR /
RUN mkdir /work-buildlinux
COPY --link --from=linux_assets /linux /work-buildlinux/linux
RUN ls /work-buildlinux
RUN ls /work-buildlinux/linux

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

FROM gcc-riscv64-linux-gnu-base AS linux-riscv64-dev-common

FROM scratch AS linux-riscv64
COPY --link --from=linux-riscv64-dev /out/Image /out/Image
COPY --link --from=linux-riscv64-config-dev /work-buildlinux/linux/.config /


