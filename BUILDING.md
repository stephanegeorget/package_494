# Building the TriCore GCC 4.9.4 Toolchain from Source

This document describes how to build the TriCore GCC cross-compiler toolchain
from this source repository on a modern Linux system.

## Overview

This repository contains three GNU components customised by HighTec (HTC) for
the Infineon TriCore AURIX architecture:

| Component    | Version | Directory   |
|-------------|---------|-------------|
| Binutils    | 2.20    | `binutils/` |
| GCC         | 4.9.4   | `gcc/`      |
| Newlib      | 1.18    | `newlib/`   |

The build follows the standard cross-compiler bootstrap sequence:

1. **Binutils** (assembler, linker, etc.)
2. **Bootstrap GCC** (C compiler only, no libc)
3. **Newlib** (C library, built with the bootstrap compiler)
4. **libgcc** (GCC runtime library, built against newlib headers)

## Prerequisites

Tested on Ubuntu/Debian with GCC 13. Required packages:

```bash
sudo apt-get install build-essential texinfo libgmp-dev libmpfr-dev libmpc-dev flex bison
```

## Repository Issues and Workarounds

Building this repo on a modern (2024+) Linux host requires several workarounds
for compatibility issues. These are documented here so they can be understood
and potentially fixed at the source.

### 1. Missing Execute Permissions (574 files)

**Problem:** All `configure` scripts, `.sh` files, and other executable scripts
(`install-sh`, `depcomp`, `missing`, `mkinstalldirs`, `move-if-change`, `ylwrap`,
`compile`, `mkdep`, `ltmain.sh`, `config.guess`, `config.sub`, `config.rpath`)
are committed to git with mode `100644` (not executable). GNU autotools requires
these to be executable.

**Cause:** The original repository was likely created on Windows or the files
were added to git without preserving the execute bit.

**Fix:** Run `chmod +x` on all affected files before building:

```bash
# In the repository root:
find binutils gcc newlib -type f \( \
    -name "configure" -o -name "*.sh" -o -name "install-sh" -o \
    -name "depcomp" -o -name "missing" -o -name "mkinstalldirs" -o \
    -name "move-if-change" -o -name "ylwrap" -o -name "compile" -o \
    -name "mkdep" -o -name "ltmain.sh" -o -name "config.guess" -o \
    -name "config.sub" -o -name "config.rpath" \
\) -exec chmod +x {} +
```

**Ideal fix:** Commit these files with the execute bit set (`git update-index --chmod=+x`).

### 2. GCC 10+ `-fno-common` Default (binutils)

**Problem:** GCC 10 changed the default from `-fcommon` to `-fno-common`. The
binutils code has a tentative definition `p_xml_element xml_root;` in the header
`binutils/include/xml.h` (line 46), which is included by multiple translation
units. With `-fno-common`, this causes "multiple definition" linker errors.

**Fix:** Pass `-fcommon` when building binutils:

```bash
CFLAGS="-fcommon -g -O2" CXXFLAGS="-fcommon -g -O2" ../binutils/configure ...
```

**Ideal fix:** Change the declaration in `xml.h` to `extern p_xml_element xml_root;`
and add the definition in exactly one `.c` file.

### 3. C++17 Removes `operator++` on `bool` (GCC source)

**Problem:** GCC 4.9.4's own source code uses `bool` increment (`spill_indirect_levels++`
in `gcc/reload1.c`), which was removed in C++17. Modern host compilers default
to C++17 or later and reject this.

**Fix:** Force C++14 when building GCC:

```bash
CXXFLAGS="-fcommon -g -O2 -std=gnu++14" ../gcc/configure ...
```

### 4. HTC License Check Fails Without Licenser Binary

**Problem:** The compiler includes an HTC proprietary license check
(`gcc/gcc/config/htc-licenser.c`) that tries to execute an `htc-licenser` binary.
This binary is not included in the source repository.

**Fix:** Set the environment variable to skip the check:

```bash
export HTC_SKIP_LICENSE_CHECK=1
```

This must be set for all build steps that invoke the cross-compiler (newlib,
libgcc builds) and when using the compiler afterwards.

### 5. License Skip Warning Breaks `-Werror` Builds

**Problem:** When `HTC_SKIP_LICENSE_CHECK` is set, the code originally emitted a
`warning()` diagnostic. The TriCore target's Makefile (`gcc/gcc/config/tricore/t-tricore`)
compiles `crt0.S` files with `-Werror`, causing this warning to be promoted to
an error, failing the libgcc/crt0 build.

**Fix:** Edit `gcc/gcc/config/htc-licenser.c` to use `inform()` instead of
`warning()`. The `inform()` function emits a "note:" level diagnostic that is
not affected by `-Werror`:

```c
// Change this (line 149):
warning (0, "skipping htc license check ...");
// To this:
inform (UNKNOWN_LOCATION, "skipping htc license check ...");
```

## Build Instructions

### Setup

```bash
# Set install prefix
export PREFIX=$(pwd)/install
export PATH="$PREFIX/bin:$PATH"
export HTC_SKIP_LICENSE_CHECK=1

# Fix permissions (see issue #1 above)
find binutils gcc newlib -type f \( \
    -name "configure" -o -name "*.sh" -o -name "install-sh" -o \
    -name "depcomp" -o -name "missing" -o -name "mkinstalldirs" -o \
    -name "move-if-change" -o -name "ylwrap" -o -name "compile" -o \
    -name "mkdep" -o -name "ltmain.sh" -o -name "config.guess" -o \
    -name "config.sub" -o -name "config.rpath" \
\) -exec chmod +x {} +

# Create out-of-tree build directories
mkdir -p build/{binutils,gcc-bootstrap,newlib}
```

### Step 1: Build Binutils

```bash
cd build/binutils
../../binutils/configure \
    --target=tricore \
    --prefix=$PREFIX \
    --disable-nls \
    --disable-werror \
    CFLAGS="-fcommon -g -O2" \
    CXXFLAGS="-fcommon -g -O2"
make -j$(nproc)
make install
cd ../..
```

### Step 2: Build Bootstrap GCC

```bash
cd build/gcc-bootstrap
../../gcc/configure \
    --target=tricore \
    --prefix=$PREFIX \
    --enable-languages=c \
    --without-headers \
    --with-newlib \
    --disable-shared \
    --disable-threads \
    --disable-libssp \
    --disable-libgomp \
    --disable-libmudflap \
    --disable-nls \
    --disable-werror \
    CFLAGS="-fcommon -g -O2" \
    CXXFLAGS="-fcommon -g -O2 -std=gnu++14"
make -j$(nproc) all-gcc
make install-gcc
cd ../..
```

### Step 3: Build Newlib

```bash
cd build/newlib
../../newlib/configure \
    --target=tricore \
    --prefix=$PREFIX \
    --disable-newlib-supplied-syscalls \
    CFLAGS_FOR_TARGET="-g -O2 -ffunction-sections"
make -j$(nproc)
make install
cd ../..
```

### Step 4: Build libgcc

```bash
cd build/gcc-bootstrap
make -j$(nproc) all-target-libgcc
make install-target-libgcc
cd ../..
```

### Verification

```bash
tricore-gcc --version
# Should output: tricore-gcc (cosmocomp Release GCC) 4.9.4

# Test compilation:
echo 'int main(void) { return 0; }' > /tmp/test.c
tricore-gcc -c /tmp/test.c -o /tmp/test.o -mcpu=tc27xx
tricore-objdump -d /tmp/test.o
# Should show TriCore assembly instructions (mov.aa, ret, etc.)
```

Note: Linking a full ELF binary requires a memory-map linker script specific to
your target chip (e.g., the ESX-4CT), which is provided by your embedded project,
not by this toolchain.

## Installed Components

After a successful build, `$PREFIX/` contains:

- `bin/` — Cross-tools: `tricore-gcc`, `tricore-as`, `tricore-ld`, `tricore-objdump`, etc.
- `lib/gcc/tricore/4.9.4/` — Compiler internals, libgcc, crt0 objects for all multilib variants
- `tricore/lib/` — Newlib libraries (libc.a, libm.a, libg.a, libos.a)
- `tricore/include/` — Newlib C headers

Multilib variants are built for: default, tc131, tc16, tc161, tc162, each with
an optional short-double variant.

## Files That Should Not Be in Version Control

The repository tracks several generated files that cause unnecessary noise in
diffs and can be regenerated during the build:

- `gcc/mpfr/autom4te.cache/` — Autoconf cache directory (regenerated by autotools)
- `gcc/gmp/doc/gmp.info` — Generated from `gmp.texi` by `makeinfo` (regenerated during build)
- `gcc/mpfr/configure` — Generated from `configure.in` by autoconf (regenerated if host autoconf version differs)

These get modified during the build process when the host system has different
versions of autotools/texinfo than what originally generated them, producing
large but meaningless diffs.
