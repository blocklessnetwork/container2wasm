SHELL := /bin/bash
all: build-linux

build-linux:
	if [[ ! -d linux ]]; then git clone -b v6.1 --depth 1 https://github.com/torvalds/linux linux; fi
	if [[ ! -d assets ]]; then git clone -b v0.6.5 https://github.com/ktock/container2wasm assets; fi
	docker build -f Dockerfile .