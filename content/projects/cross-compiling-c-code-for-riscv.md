---
title: "Cross Compiling C Code for RISC-V"
date: 2025-06-04T17:45:46+05:30
slug: cross-compiling-c-code-for-riscv
category: projects
summary:
description:
cover:
  image: /images/projects/cross-compiling-c-code-for-riscv.jpeg
  
  alt:
  caption:
  relative: true
showtoc: true
draft: false
---

# Cross-Compiling C Code for RISC-V (with Vector Extensions) and Performance Comparison

Harshil Kaneria and I worked under Dr. Binod Kumar where we assisted Jaitra.inc to build a Cross-Compiling C Code for RISC-V. Our work was to execute [llama2.c](https://github.com/karpathy/llama2.c) file in risc-v and check its performance. 



This guide covers the process of cross-compiling C programs for RISC-V 64-bit Linux targets using LLVM/Clang and GNU toolchains. It also includes performance benchmarks across different optimization levels and platforms. 

---

## Prerequisites

To build and test RISC-V binaries, you need two main toolchains:

* **LLVM/Clang (for Vector Extensions)**: For experimental support and vector intrinsics.
* **RISC-V GNU Toolchain**: For linking and library support.

---

## Installation Steps

### 1. Clone Required Repositories

*Clone the official RISC-V GNU toolchain repository to get started with building the compiler.*

```bash
git clone https://github.com/riscv/riscv-gnu-toolchain
```

### 2. Install Build Dependencies

*Install all required development tools and libraries to successfully build both LLVM and GNU toolchains.*

```bash
sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip \
libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo \
gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake \
libglib2.0-dev libslirp-dev libfdt-dev libpixman-1-dev
```

### 3. Set Up the Environment

*Create a local installation directory and add it to your PATH to access the compiled toolchains easily.*

This step sets up the `_install` directory where all binaries and libraries will be installed. The `export PATH=...` line ensures that your shell can recognize the newly built binaries by prepending this path. `hash -r` refreshes the shell’s internal command lookup cache.

**If at any point your terminal doesn't recognize the `riscv64-unknown-linux-gnu-*` commands, re-run the `export PATH=...` line to ensure your environment is correctly configured.**

```bash
mkdir _install
export PATH=`pwd`/_install/bin:$PATH
hash -r
```

### 4. Clone LLVM Toolchain

*Fetch the latest LLVM project source code, which includes Clang and related tools.*

```bash
git clone https://github.com/llvm/llvm-project.git riscv-llvm
```

### 5. Build GNU Toolchain and QEMU

*Build the RISC-V GNU cross-compiler and optionally QEMU for simulating RISC-V binaries.*

The `configure` step sets the installation path and enables multilib support for multiple ABI/ISA variants. `make linux` builds the Linux-targeted compiler. `make build-qemu` builds the QEMU emulator which is useful for testing your binaries on non-native hardware.

```bash
pushd riscv-gnu-toolchain
./configure --prefix=`pwd`/../_install --enable-multilib
make linux -j`nproc`
make -j`nproc` build-qemu
popd
```

### 6. Build LLVM Toolchain for RISC-V

*Configure and build LLVM with RISC-V as the target architecture using CMake and Ninja.*

* `CMAKE_BUILD_TYPE=Release`: Builds LLVM in optimized release mode.
* `BUILD_SHARED_LIBS=True`: Enables building shared libraries instead of static ones.
* `LLVM_USE_SPLIT_DWARF=True`: Splits debug info into separate files for faster build and smaller binaries.
* `CMAKE_INSTALL_PREFIX=...`: Specifies the output directory (`_install`).
* `LLVM_OPTIMIZED_TABLEGEN=True`: Builds the TableGen tool in optimized mode.
* `LLVM_BUILD_TESTS=False`: Skips test builds for faster compilation.
* `DEFAULT_SYSROOT=...`: Points LLVM to use libraries from the previously installed GNU toolchain.
* `LLVM_DEFAULT_TARGET_TRIPLE=riscv64-unknown-linux-gnu`: Sets RISC-V as the default compilation target.
* `LLVM_TARGETS_TO_BUILD=RISCV`: Only builds the RISC-V backend to reduce build time.

These flags are critical for ensuring LLVM properly supports RISC-V and integrates well with the GNU libraries.
Refer to the [official LLVM CMake options documentation](https://llvm.org/docs/CMake.html) for more details.

```bash
pushd riscv-llvm
ln -s ../../clang llvm/tools || true
mkdir _build && cd _build

cmake -G Ninja -DCMAKE_BUILD_TYPE="Release" \
  -DBUILD_SHARED_LIBS=True -DLLVM_USE_SPLIT_DWARF=True \
  -DCMAKE_INSTALL_PREFIX="../../_install" \
  -DLLVM_OPTIMIZED_TABLEGEN=True -DLLVM_BUILD_TESTS=False \
  -DDEFAULT_SYSROOT="../../_install/sysroot" \
  -DLLVM_DEFAULT_TARGET_TRIPLE="riscv64-unknown-linux-gnu" \
  -DLLVM_TARGETS_TO_BUILD="RISCV" \
  ../llvm

cmake --build . --target install
popd
```

---

## Sanity Check :)

Compile and run a sample C program on QEMU:

```c
// hello.c
#include <stdio.h>
int main() {
    printf("Hello RISCV!\n");
    return 0;
}
```

```bash
clang -O -c hello.c
riscv64-unknown-linux-gnu-gcc hello.o -o hello -march=rv64gc -mabi=lp64d
qemu-riscv64 hello
```

---

## 🧪 Benchmarking (Llama and Ngram)

Since we tested this setup with [llama2.c](https://github.com/karpathy/llama2.c), here’s how you can compile it:

### Go to the llama repo cloned from GitHub, follow its steps, and at time of compilation use:

```bash
clang -O3 -c run.c
riscv64-unknown-linux-gnu-gcc run.o -o run -march=rv64gc -mabi=lp64d -lm
```

### Test Platform: RISC-V 64-bit Board (8GB RAM, 4 cores)

#### Optimization Levels Comparison (tokens/sec)

##### Llama 15M

| Optimization | Clang + GCC | GCC   | Clang |
| ------------ | ----------- | ----- | ----- |
| O            | 13.11       | 8.05  | 13.16 |
| O1           | 13.05       | 7.85  | 13.30 |
| O2           | 13.39       | 13.19 | 12.90 |
| O3           | 13.32       | 13.19 | 13.01 |
| Ofast        | 13.31       | 13.26 | 13.11 |

##### Llama 42M

| Optimization | Clang + GCC | GCC  | Clang |
| ------------ | ----------- | ---- | ----- |
| O            | 5.13        | 2.98 | 5.20  |
| O1           | 5.12        | 3.03 | 5.20  |
| O2           | 5.11        | 5.21 | 5.22  |
| O3           | 5.23        | 5.22 | 5.15  |
| Ofast        | 5.19        | 5.21 | 5.20  |

---

## 🧪 Comparison on Other Platforms

### Raspberry Pi 4 (ARM64, 4GB RAM)

#### Llama 15M (tokens/sec)

| Optimization | GCC   | Clang |
| ------------ | ----- | ----- |
| O            | 29.16 | 28.36 |
| O1           | 28.76 | 27.95 |
| O2           | 29.62 | 28.45 |
| O3           | 27.24 | 28.59 |
| Ofast        | 34.92 | 34.31 |

---

### x86-64 POP OS (128GB RAM, 64 Cores)

#### Llama 15M (tokens/sec)

| Optimization | Clang  | GCC    |
| ------------ | ------ | ------ |
| O            | 86.22  | 79.20  |
| O1           | 85.50  | 78.44  |
| O2           | 86.51  | 84.21  |
| O3           | 89.20  | 89.59  |
| Ofast        | 322.58 | 315.63 |

---

## ⏱ Execution Time Metrics (RISC-V 64-bit)

### Llama 15M

| Optimization | Clang + GCC (real) | GCC (real) | Clang (real) |
| ------------ | ------------------ | ---------- | ------------ |
| O            | 0m16.73s           | 0m21.62s   | 0m16.58s     |
| O1           | 0m22.28s           | 0m28.53s   | 0m14.45s     |
| O2           | 0m13.42s           | 0m10.05s   | 0m15.32s     |
| O3           | 0m17.73s           | 0m16.73s   | 0m17.86s     |
| Ofast        | 0m13.68s           | 0m9.70s    | 0m19.59s     |

---

### Ngram.c on Raspberry Pi (time)

| Optimization | GCC (real) | Clang (real) |
| ------------ | ---------- | ------------ |
| O            | 0.058s     | 0.048s       |
| O1           | 0.058s     | 0.048s       |
| O2           | 0.054s     | 0.043s       |
| O3           | 0.050s     | 0.047s       |
| Ofast        | 0.049s     | 0.046s       |


---

For more details, contributions, or issues, visit:

* [LLVM Project](https://github.com/llvm/llvm-project)
* [RISC-V GNU Toolchain](https://github.com/riscv/riscv-gnu-toolchain)

