# Rust Linux / Darwin Builder [![Docker Image Version (tag latest semver)](https://img.shields.io/docker/v/joseluisq/rust-linux-darwin-builder/1)](https://hub.docker.com/r/joseluisq/rust-linux-darwin-builder/) [![Build Status](https://travis-ci.com/joseluisq/rust-linux-darwin-builder.svg?branch=master)](https://travis-ci.com/joseluisq/rust-linux-darwin-builder) [![Docker Image Size (tag)](https://img.shields.io/docker/image-size/joseluisq/rust-linux-darwin-builder/1)](https://hub.docker.com/r/joseluisq/rust-linux-darwin-builder/tags) [![Docker Image](https://img.shields.io/docker/pulls/joseluisq/rust-linux-darwin-builder.svg)](https://hub.docker.com/r/joseluisq/rust-linux-darwin-builder/)

> Use same Docker image for compiling [Rust](https://www.rust-lang.org/) programs for Linux ([musl libc](https://doc.rust-lang.org/edition-guide/rust-2018/platform-and-target-support/musl-support-for-fully-static-binaries.html)) & macOS ([osxcross](https://github.com/tpoechtrager/osxcross)).

## OpenSSL Release Notes

Until `v1.42.0` of this project, one old OpenSSL release `v1.0.2` was used. 
Now, since `v1.43.x` or greater, OpenSSL `v1.1.1` (LTS) is used which is supported until `2023-09-11`. View more at https://www.openssl.org/policies/releasestrat.html.

## Overview

This is a __Linux Docker image__ based on [ekidd/rust-musl-builder](https://hub.docker.com/r/ekidd/rust-musl-builder) but using [debian:buster-slim](https://hub.docker.com/_/debian?tab=tags&page=1&name=buster-slim). It contains essential tools for compile [Rust](https://www.rust-lang.org/) projects such as __Linux__ static binaries via [musl-libc / musl-gcc](https://doc.rust-lang.org/edition-guide/rust-2018/platform-and-target-support/musl-support-for-fully-static-binaries.html) (`x86_64-unknown-linux-musl`) and __macOS__ binaries (`x86_64-apple-darwin`) via [osxcross](https://github.com/tpoechtrager/osxcross) just using the same Linux image.

## Usage

### Compiling an application inside a Docker container

By default the working directory is `/root/src`.

#### x86_64-unknown-linux-musl

```sh
docker run --rm \
    --volume "${PWD}/sample":/root/src \
    --workdir /root/src \
    joseluisq/rust-linux-darwin-builder:1.50.0 \
    sh -c "cargo build --release"
```

#### x86_64-apple-darwin

```sh
docker run --rm \
    --volume "${PWD}/sample":/root/src \
    --workdir /root/src \
    joseluisq/rust-linux-darwin-builder:1.50.0 \
    sh -c "cargo build --release --target x86_64-apple-darwin"
```

### Dockerfile

You can also use the image as a base for your own Dockerfile:

```Dockerfile
FROM joseluisq/rust-linux-darwin-builder:1.50.0
```

### OSXCross

You can also use o32-clang(++) and o64-clang(++) like a normal compiler.

__Notes:__

- The current *11.1 SDK* does not support i386 anymore. Use <= 10.13 SDK if you rely on i386 support.
- The current *11.1 SDK* does not support libstdc++ anymore. Use <= 10.13 SDK if you rely on libstdc++ support.

Examples:

```sh
Example usage:

Example 1: CC=o32-clang ./configure --host=i386-apple-darwin20.2
Example 2: CC=i386-apple-darwin20.2-clang ./configure --host=i386-apple-darwin20.2
Example 3: o64-clang -Wall test.c -o test
Example 4: x86_64-apple-darwin20.2-strip -x test

!!! Use aarch64-apple-darwin20.2-* instead of arm64-* when dealing with Automake !!!
!!! CC=aarch64-apple-darwin20.2-clang ./configure --host=aarch64-apple-darwin20.2 !!!
!!! CC="aarch64-apple-darwin20.2-clang -arch arm64e" ./configure --host=aarch64-apple-darwin20.2 !!!
```

### Cross-compilation example

Below a simple example using a `Makefile` for cross-compiling a Rust app.

Notes:

- A [hello world](./tests/hello-world) app is used.
- A custom directory is used below as working directory instead of `/root/src`.

Create a Makefile:

```sh
compile:
	@docker run --rm -it \
		-v $(PWD):/drone/src \
		-w /drone/src \
			joseluisq/rust-linux-darwin-builder:1.45.2 \
				make cross-compile
.PHONY: compile

cross-compile:
	@echo
	@echo "1. Cross compiling example..."
	@rustc -vV
	@echo
	@echo "2. Compiling application (linux-musl x86_64)..."
	@cargo build --manifest-path=tests/hello-world/Cargo.toml --release --target x86_64-unknown-linux-musl
	@du -sh tests/hello-world/target/x86_64-unknown-linux-musl/release/helloworld
	@echo
	@echo "3. Compiling application (apple-darwin x86_64)..."
	@cargo build --manifest-path=tests/hello-world/Cargo.toml --release --target x86_64-apple-darwin
	@du -sh tests/hello-world/target/x86_64-apple-darwin/release/helloworld
.PHONY: cross-compile
```

Just run the makefile `compile` target, then you will see two release binaries `x86_64-unknown-linux-musl` and `x86_64-apple-darwin`).

```sh
make compile
# 1. Cross compiling example...

# rustc 1.45.2 (d3fb005a3 2020-07-31)
# binary: rustc
# commit-hash: d3fb005a39e62501b8b0b356166e515ae24e2e54
# commit-date: 2020-07-31
# host: x86_64-unknown-linux-gnu
# release: 1.45.2
# LLVM version: 10.0

# 2. Compiling application (linux-musl x86_64)...
#     Finished release [optimized] target(s) in 0.01s
# 1.2M	tests/hello-world/target/x86_64-unknown-linux-musl/release/helloworld

# 3. Compiling application (apple-darwin x86_64)...
#     Finished release [optimized] target(s) in 0.01s
# 240K	tests/hello-world/target/x86_64-apple-darwin/release/helloworld
```

### Building *-sys crates

If some of your crates require C bindings and you run into compilation or linking errors, try to use Clang for C/C++ builds.

For example to cross compile to Macos:

```sh
CC=o64-clang \
CXX=o64-clang++ \
	cargo build --target x86_64-apple-darwin
```

Read more details about it take a look at [Cross compiling Rust from Linux to macOS](https://wapl.es/rust/2019/02/17/rust-cross-compile-linux-to-macos.html) by James Waples.

## Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in current work by you, as defined in the Apache-2.0 license, shall be dual licensed as described below, without any additional terms or conditions.

Feel free to send some [Pull request](https://github.com/joseluisq/rust-linux-darwin-builder/pulls) or [issue](https://github.com/joseluisq/rust-linux-darwin-builder/issues).

## License

This work is primarily distributed under the terms of both the [MIT license](LICENSE-MIT) and the [Apache License (Version 2.0)](LICENSE-APACHE).

© 2019-present [Jose Quintana](https://git.io/joseluisq)
