ARG OPTIMIZATION_MODE=wizer
ARG INIT_DEBUG=true
ARG NO_VMTOUCH=
ARG EXTERNAL_BUNDLE=
ARG TARGETPLATFORM=amd64

FROM scratch as assets
COPY . .

FROM ubuntu:22.04 as bundle-dev
ARG INIT_DEBUG
ARG OPTIMIZATION_MODE
ARG NO_VMTOUCH
ARG EXTERNAL_BUNDLE
ARG TARGETPLATFORM
WORKDIR /work
COPY --link --from=assets / /oci
COPY --link --from=joing/bls-c2w-tools /tools /tools
RUN mkdir -p /out/oci/rootfs /out/oci/bundle && \
    IS_WIZER=false && \
    if test "${OPTIMIZATION_MODE}" = "wizer" ; then IS_WIZER=true ; fi && \
    NO_VMTOUCH_F=false && \
    if test "${OPTIMIZATION_MODE}" = "native" ; then NO_VMTOUCH_F=true ; fi && \
    if test "${NO_VMTOUCH}" != "" ; then NO_VMTOUCH_F="${NO_VMTOUCH}" ; fi && \
    EXTERNAL_BUNDLE_F=false && \
    if test "${EXTERNAL_BUNDLE}" = "true" ; then EXTERNAL_BUNDLE_F=true ; fi && \
    /tools/create-spec --debug=${INIT_DEBUG} --debug-init=${IS_WIZER} --no-vmtouch=${NO_VMTOUCH_F} --external-bundle=${EXTERNAL_BUNDLE_F} \
                --image-config-path=/oci/image.json \
                --runtime-config-path=/oci/spec.json \
                --rootfs-path=/oci/rootfs \
                /oci "${TARGETPLATFORM}" /out/oci/rootfs
RUN if test -f image.json; then mv image.json /out/oci/ ; fi && \
    if test -f spec.json; then mv spec.json /out/oci/ ; fi
RUN mv initconfig.json /out/oci/


FROM ubuntu:22.04 AS rootfs-amd64-dev
RUN apt-get update -y && apt-get install -y mkisofs
COPY --link --from=joing/bls-amd64-linux6.1 /rootfs /rootfs
COPY --link --from=bundle-dev /out/ /rootfs/
RUN mkdir -p /rootfs/proc /rootfs/sys /rootfs/mnt /rootfs/run /rootfs/tmp /rootfs/dev /rootfs/var /rootfs/etc && mknod /rootfs/dev/null c 1 3 && chmod 666 /rootfs/dev/null
RUN mkdir /out/ && mkisofs -l -J -R -o /out/rootfs.bin /rootfs/

FROM scratch AS vm-amd64-dev
COPY --link --from=joing/bls-amd64-linux6.1 /pack /pack
COPY --link --from=rootfs-amd64-dev /out/rootfs.bin /pack


FROM ubuntu:22.04 AS dev-packed
ENV WASMTIME_BACKTRACE_DETAILS=1
COPY --link --from=joing/bls-c2w-tools /tools /tools
COPY --link --from=vm-amd64-dev /pack /pack
COPY --link --from=joing/bls-tinyemu-wasi-19 /out/bochs /work/bochs
WORKDIR /work

RUN mv bochs bochs-org && /tools/wizer/wizer-debug --allow-wasi --wasm-bulk-memory=true -r _start=wizer.resume --mapdir /pack::/pack -o bochs bochs-org
RUN mkdir /minpack && cp /pack/rootfs.bin /minpack/ && cp /pack/boot.iso /minpack/

RUN /tools/wasi-vfs/wasi-vfs pack bochs --mapdir /pack::/minpack -o packed && mkdir /out && mv packed /out

FROM scratch AS wasi-amd64
ARG OUTPUT_NAME
COPY --link --from=dev-packed /out/packed /${OUTPUT_NAME}


FROM wasi-amd64