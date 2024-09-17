SHELL := /bin/bash

WASI_SDK_VERSION=19
WASI_SDK_VERSION_FULL=${WASI_SDK_VERSION}.0
WASI_VFS_VERSION=v0.3.0
BINARYEN_VERSION=114
BOCHS_REPO=https://github.com/ktock/Bochs
BOCHS_REPO_VERSION=a88d1f687ec83ff82b5318f59dcecb8dab44fc83
TINYEMU_REPO=git@github.com:ktock/tinyemu-c2w.git
TINYEMU_REPO_VERSION=e4e9bd198f9c0505ab4c77a6a9d038059cd1474a
WIZER_VERSION=04e49c989542f2bf3a112d60fbf88a62cce2d0d0

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

	if [[ ! -d wasi-vfs ]]; then \
		git clone ${BOCHS_REPO} /Bochs &&\
		cd /Bochs &&\
		git checkout ${BOCHS_REPO_VERSION} \
	; fi

	if [[ ! -f wizer.h ]]; then \
		curl -o wizer.h -L https://raw.githubusercontent.com/bytecodealliance/wizer/04e49c989542f2bf3a112d60fbf88a62cce2d0d0/include/wizer.h \
	; fi

	if [[ ! -d tinyemu ]]; then \
		git clone ${TINYEMU_REPO} tinyemu  &&\
		cd tinyemu &&\
		git checkout ${TINYEMU_REPO_VERSION} \
	; fi

	if [[ ! -d Bochs ]]; then \
		git clone ${BOCHS_REPO} Bochs  &&\
		cd Bochs &&\
		git checkout ${BOCHS_REPO_VERSION} \
	; fi
	if [[ ! -f binaryen.tar.gz ]]; then \
		curl -o binaryen.tar.gz -L https://github.com/WebAssembly/binaryen/releases/download/version_${BINARYEN_VERSION}/binaryen-version_${BINARYEN_VERSION}-x86_64-linux.tar.gz \
	; fi
	

	
	docker build -t joing/bls-tinyemu-wasi-19  -f Dockerfile .
push:
	docker push joing/bls-tinyemu-wasi-19
