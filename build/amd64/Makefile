SHELL := /bin/bash
GRUB_VERSION=2.06
BUSYBOX_VERSION=1.36.1
IMAGE_NAME=bls-amd64-linux6.1

all: build

build:
	if [[ ! -f linux.tar.gz ]]; then \
		curl -Lo linux.tar.gz https://github.com/torvalds/linux/archive/refs/tags/v6.1.tar.gz \
	; fi
	if [[ ! -d assets ]]; then git clone -b v0.6.5 https://github.com/ktock/container2wasm assets; fi
	if [[ ! -f grub-${GRUB_VERSION}.tar.gz ]]; then \
		curl -Lo grub-${GRUB_VERSION}.tar.gz https://ftp.gnu.org/gnu/grub/grub-${GRUB_VERSION}.tar.gz \
	; fi
	if [[ ! -f busybox-${BUSYBOX_VERSION}.tar.bz ]]; then \
		curl -Lo busybox-${BUSYBOX_VERSION}.tar.bz https://busybox.net/downloads/busybox-${BUSYBOX_VERSION}.tar.bz2 \
	; fi
	if [[ ! -f runc.tar.gz ]]; then \
		curl -Lo runc.tar.gz https://github.com/opencontainers/runc/archive/refs/tags/v1.1.12.tar.gz \
	; fi
	if [[ ! -f tini.tar.gz ]]; then \
		curl -Lo tini.tar.gz https://github.com/krallin/tini/archive/refs/tags/v0.19.0.tar.gz \
	; fi
	if [[ ! -f vmtouch.tar.gz ]]; then \
		curl -Lo vmtouch.tar.gz https://github.com/hoytech/vmtouch/archive/refs/tags/v1.3.1.tar.gz \
	; fi
	docker build -t joing/${IMAGE_NAME}  -f Dockerfile .
	
push:
	docker push joing/${IMAGE_NAME}

clean:
	rm -rf grub-${GRUB_VERSION}.tar.gz