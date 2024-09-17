SHELL := /bin/bash

WASI_SDK_VERSION=19
WASI_SDK_VERSION_FULL=${WASI_SDK_VERSION}.0
WASI_VFS_VERSION=v0.3.0

all: build

build:
	if [[ ! -f wasi-sdk.tar.gz ]]; then \
		curl -o wasi-sdk.tar.gz -fSL https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-${WASI_SDK_VERSION}/wasi-sdk-${WASI_SDK_VERSION_FULL}-linux.tar.gz \
	; fi

	if [[ ! -d wasi-vfs ]]; then \
		git clone https://github.com/kateinoigakukun/wasi-vfs.git --recurse-submodules &&\
		cd wasi-vfs &&\
		git checkout "${WASI_VFS_VERSION}" \
	; fi
	
	docker build  -f Dockerfile .