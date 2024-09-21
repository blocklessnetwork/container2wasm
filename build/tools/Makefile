SHELL := /bin/bash
WIZER_VFS_VERSION=v0.3.0
IMAGE_NAME=bls-c2w-tools
WIZER_VERSION=04e49c989542f2bf3a112d60fbf88a62cce2d0d0

all: build

build:
	if [[ ! -d assets ]]; then git clone -b v0.6.5 https://github.com/ktock/container2wasm assets; fi
	if [[ ! -d wasi-vfs ]]; then \
		git clone https://github.com/kateinoigakukun/wasi-vfs.git --recurse-submodules &&\
		cd wasi-vfs &&\
		git checkout "${WIZER_VFS_VERSION}" \
	; fi
	if [[ ! -d wizer ]]; then \
		git clone https://github.com/bytecodealliance/wizer &&\
		cd wizer &&\
		git checkout "${WIZER_VERSION}" \
	; fi
	
	docker build -t joing/${IMAGE_NAME}  -f Dockerfile . --progress=plain

push:
	docker push joing/${IMAGE_NAME}

clean:
	rm -rf wizer