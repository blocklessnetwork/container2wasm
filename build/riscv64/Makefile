SHELL := /bin/bash
all: build-linux

build-linux:
	if [[ ! -f linux.tar.gz ]]; then \
		curl -Lo linux.tar.gz https://github.com/torvalds/linux/archive/refs/tags/v6.1.tar.gz \
	; fi
	if [[ ! -d assets ]]; then git clone -b v0.6.5 https://github.com/ktock/container2wasm assets; fi
	if [[ ! -d busybox ]]; then \
		curl -Lo busybox-1.36.1.tar.bz https://busybox.net/downloads/busybox-1.36.1.tar.bz2 \
	; fi
	docker build -f Dockerfile .