SHELL := /bin/bash
all: build-linux

build-linux:
	if [[ ! -f linux.tar.gz ]]; then \
		curl -Lo linux.tar.gz https://github.com/torvalds/linux/archive/refs/tags/v6.1.tar.gz \
	; fi
	if [[ ! -d assets ]]; then git clone -b v0.6.5 https://github.com/ktock/container2wasm assets; fi
	if [[ ! -f busybox-1.36.1.tar.bz ]]; then \
		curl -Lo busybox-1.36.1.tar.bz https://busybox.net/downloads/busybox-1.36.1.tar.bz2 \
	; fi

	if [[ ! -f tini.tar.gz ]]; then \
		curl -Lo tini.tar.gz https://github.com/krallin/tini/archive/refs/tags/v0.19.0.tar.gz \
	; fi
	if [[ ! -f runc.tar.gz ]]; then \
		curl -Lo runc.tar.gz https://github.com/opencontainers/runc/archive/refs/tags/v1.1.12.tar.gz \
	; fi
	if [[ ! -f vmtouch.tar.gz ]]; then \
		curl -Lo vmtouch.tar.gz https://github.com/hoytech/vmtouch/archive/refs/tags/v1.3.1.tar.gz \
	; fi

	if [[ ! -d riscv-pk ]]; then \
		git clone https://github.com/riscv-software-src/riscv-pk && cd riscv-pk && git checkout 7e9b671c0415dfd7b562ac934feb9380075d4aa2 \
	; fi
	docker build --build-arg TARGETARCH=amd64 -t joing/blockless-riscv64-linux6.1 -f Dockerfile .

push:
	docker push joing/blockless-riscv64-linux6.1
clean:
	rm -rf assets  busybox-1.36.1.tar.bz  linux.tar.gz  riscv-pk  runc.tar.gz  tini.tar.gz  vmtouch.tar.gz
