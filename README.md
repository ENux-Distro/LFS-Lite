# LFS-Lite
### Build a Minimal Linux System and Boot It as an ISO Image

---

> **LFS-Lite** is a condensed, practical guide inspired by Linux From Scratch.
> Instead of building a full POSIX userland, you will build only the essential
> components needed for a real, bootable system — compiled entirely from source,
> using a GCC toolchain you build yourself.
> The result is a hybrid Legacy/EFI ISO image you can run in a VM or burn to a USB drive.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Setting Up the Build Environment](#3-setting-up-the-build-environment)
4. [Installing Linux API Headers](#4-installing-linux-api-headers)
5. [Building GCC (Cross Toolchain)](#5-building-gcc-cross-toolchain)
6. [Building Musl (C Library)](#6-building-musl-c-library)
7. [Building Bash](#7-building-bash)
8. [Building Coreutils](#8-building-coreutils)
9. [Building Util-Linux](#9-building-util-linux)
10. [Building the Linux Kernel](#10-building-the-linux-kernel)
11. [Building GRUB](#11-building-grub)
12. [Assembling the Root Filesystem](#12-assembling-the-root-filesystem)
13. [Creating the Bootable ISO](#13-creating-the-bootable-iso)
14. [Testing in QEMU](#14-testing-in-qemu)
15. [Package Versions Reference](#15-package-versions-reference)

---

## 1. Introduction

### What Is LFS-Lite?

Linux From Scratch (LFS) is a legendary project that teaches you to build a complete
Linux system from source code. It is thorough, educational, and takes days to complete.

LFS-Lite is for people who understand *why* LFS exists but want a shorter path to a
real, working result. You will build exactly seven components — including GCC itself:

| Component      | Role                                       |
|----------------|--------------------------------------------|
| Linux Headers  | Kernel API definitions for userspace       |
| GCC 15.2.0     | The C compiler used to build everything    |
| musl 1.2.6     | The C standard library                     |
| Bash 5.3       | The shell / PID 1 init                     |
| Coreutils 9.10 | Core POSIX utilities (ls, cp, cat, …)      |
| Util-Linux 2.42| Essential system tools (mount, fdisk, …)   |
| Linux 7.0      | The kernel                                 |
| GRUB 2.14      | The bootloader (BIOS + EFI)                |

The output is a bootable ISO image. Everything is compiled from source on your host
machine, then assembled into an initramfs and a GRUB-bootable ISO.

### What You Will Learn

- How each component fits together in a Linux system
- Why Linux headers must be installed before the C library
- How to build GCC from source and use it as your system compiler
- How to pass `./configure` flags for a minimal, musl-based build
- How to build a Linux kernel from scratch
- How to create an initramfs and wrap it into a bootable ISO with GRUB

### What This Guide Is Not

- A full LFS replacement — you will not get a package manager, init system, or a
  rich userland
- A full cross-compilation tutorial — we build GCC targeting the host arch and use
  it to compile into a separate sysroot
- A production system — this is for learning

### Build Order Rationale

The build order matters. Each package depends on the previous:

```
Linux Headers → GCC → musl → Bash → Coreutils → Util-Linux → Kernel → GRUB → ISO
```

Linux headers must come first because musl's headers reference kernel types and
constants. GCC must come before musl so we have a compiler that can target our
sysroot cleanly. Everything else follows from there.

---

## 2. Prerequisites

### Host Tools Required

Your host system (any Linux distro) must have the following **host** tools available
before starting. These are only used to bootstrap the build — they will not end up
in the final ISO.

```
gcc (host)    g++ (host)    make          bison
flex          gawk          texinfo       help2man
pkg-config    libssl-dev    xorriso       mtools
libelf-dev    libncurses-dev              python3
libgmp-dev    libmpfr-dev   libmpc-dev    m4
autoconf      automake      libtool       wget
```

**Debian/Ubuntu:**
```bash
sudo apt install build-essential bison flex gawk texinfo help2man \
    pkg-config libssl-dev xorriso mtools libelf-dev libncurses-dev \
    python3 libgmp-dev libmpfr-dev libmpc-dev m4 autoconf automake \
    libtool wget
```

**Arch Linux:**
```bash
sudo pacman -S base-devel bison flex gawk texinfo help2man \
    pkg-config openssl xorriso mtools libelf ncurses python \
    gmp mpfr libmpc m4 autoconf automake libtool wget
```

**Fedora/RHEL:**
```bash
sudo dnf install gcc gcc-c++ make bison flex gawk texinfo help2man \
    pkgconfig openssl-devel xorriso mtools elfutils-libelf-devel \
    ncurses-devel python3 gmp-devel mpfr-devel libmpc-devel m4 \
    autoconf automake libtool wget
```

> **Note:** GMP, MPFR, and MPC are required to build GCC from source. Make sure
> they are installed with their development headers (`-dev` / `-devel` packages).

### Disk Space

You will need approximately **8 GB** of free disk space:
- ~5 GB for extracted source trees and build artifacts (GCC alone is large)
- ~500 MB for the final sysroot
- ~200 MB for the ISO image

### Architecture

This guide targets **x86-64 (amd64)**. The resulting ISO will be hybrid BIOS/EFI bootable.

---

## 3. Setting Up the Build Environment

### 3.1 Directory Structure

We will use a consistent directory layout throughout this guide. All work happens
inside `$LFSLITE`, which acts as our sysroot — the root of the new system.

```bash
export LFSLITE=/opt/lfs-lite
export LFSLITE_SRC=$LFSLITE/sources
export LFSLITE_TOOLS=$LFSLITE/tools

mkdir -pv $LFSLITE/{bin,lib,lib64,usr/{bin,lib,include},etc,dev,proc,sys,run,tmp,root}
mkdir -pv $LFSLITE_SRC
mkdir -pv $LFSLITE_TOOLS
```

Persist these variables for the duration of the build. Add them to your `~/.bashrc`
or paste them into every new terminal before resuming work:

```bash
export LFSLITE=/opt/lfs-lite
export LFSLITE_SRC=$LFSLITE/sources
export LFSLITE_TOOLS=$LFSLITE/tools
export PATH=$LFSLITE_TOOLS/bin:$PATH
```

### 3.2 Downloading Sources

Download all source tarballs into `$LFSLITE_SRC`. See [Section 15](#15-package-versions-reference)
for the exact versions used in this guide.

```bash
cd $LFSLITE_SRC

# Linux kernel (used for both headers and the final kernel)
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.tar.xz

# GCC toolchain
wget https://ftp.gnu.org/gnu/gcc/gcc-15.2.0/gcc-15.2.0.tar.xz

# C library
wget https://musl.libc.org/releases/musl-1.2.6.tar.gz

# Shell and userland
wget https://ftp.gnu.org/gnu/bash/bash-5.3.tar.gz
wget https://ftp.gnu.org/gnu/coreutils/coreutils-9.10.tar.xz
wget https://www.kernel.org/pub/linux/utils/util-linux/v2.42/util-linux-2.42.tar.xz

# Bootloader
wget https://ftp.gnu.org/gnu/grub/grub-2.14.tar.xz
```

Verify the downloads:

```bash
ls -lh $LFSLITE_SRC
```

### 3.3 Build Flags

Throughout this guide we use a consistent set of compiler flags:

```bash
export CFLAGS="-Os -pipe"
export CXXFLAGS="-Os -pipe"
export MAKEFLAGS="-j$(nproc)"
```

`-Os` optimizes for size rather than speed — appropriate for a minimal system.
`-j$(nproc)` parallelizes the build across all available CPU cores.

### 3.4 Target Triplet

We define the target triplet once and reference it everywhere:

```bash
export LFSLITE_TARGET="x86_64-linux-musl"
export LFSLITE_HOST="x86_64-linux-gnu"
```

This triplet tells GCC and other tools that we are building for a 64-bit x86 Linux
system that uses musl as its C library.

---

## 4. Installing Linux API Headers

### Why Headers Come First

The Linux kernel exports a stable set of types, constants, and system call interfaces
that userspace programs depend on. These are called the **Linux API headers** (sometimes
called UAPI headers). musl's own headers include them internally — for example,
`<sys/types.h>` pulls in kernel definitions for `pid_t`, `uid_t`, and so on.

If musl is built without kernel headers available, it cannot correctly define these
types, and the resulting C library will be broken or refuse to compile entirely.
Headers must therefore be installed before musl — and before GCC, since GCC's
libgcc also consults them.

### 4.1 Extract the Kernel Sources

```bash
cd $LFSLITE_SRC
tar xf linux-7.0.tar.xz
cd linux-7.0
```

### 4.2 Install the Headers

The kernel's build system has a dedicated `headers_install` target that sanitizes
the raw kernel headers into a clean, userspace-safe form. We install them directly
into our sysroot:

```bash
make mrproper

make headers_install \
    ARCH=x86_64 \
    INSTALL_HDR_PATH=$LFSLITE/usr
```

**What `mrproper` does:** Cleans any stale build configuration or generated files.
Always run this before `headers_install` to avoid accidentally including leftover
build artifacts.

**What `headers_install` does:** Runs the kernel's header sanitizer, which strips
`#ifdef __KERNEL__` guards and removes kernel-internal definitions, leaving only
the stable ABI that userspace is allowed to see. The result is installed under
`$LFSLITE/usr/include/`.

### 4.3 Verify

```bash
ls $LFSLITE/usr/include/
# You should see: asm  asm-generic  drm  linux  misc  mtd  rdma  scsi  sound  video  xen
```

Spot-check a key header:

```bash
head -5 $LFSLITE/usr/include/linux/types.h
# Should show: #ifndef _LINUX_TYPES_H ...
```

> **Do not modify these headers.** They are generated by the kernel's own tooling
> and represent the exact interface the kernel exposes. Editing them by hand causes
> subtle, hard-to-debug breakage downstream.

---

## 5. Building GCC (Cross Toolchain)

We build GCC 15.2.0 from source and install it into `$LFSLITE_TOOLS`. This compiler
will be used to build every subsequent package in this guide. Building GCC is the
longest step — expect 20–60 minutes depending on your hardware.

We perform a **two-stage build**:

- **Stage 1:** Build a minimal GCC configured with `--with-newlib --without-headers`
  and `--disable-shared`. This tells GCC not to expect a real libc yet. We only
  compile `gcc` and `libgcc` — just enough to compile musl.
- **Stage 2:** After musl is installed (Section 6.4), we do a **full fresh
  reconfigure and rebuild** of GCC — this time without the `--with-newlib`,
  `--without-headers`, and `--disable-shared` restrictions, and with musl's headers
  and library properly in the sysroot. Stage 2 produces the complete, production-quality
  compiler (including `libstdc++`, `libgcc_s`, and shared library support) that we
  use for all remaining packages.

### 5.1 Extract GCC

```bash
cd $LFSLITE_SRC
tar xf gcc-15.2.0.tar.xz
cd gcc-15.2.0
```

### 5.2 Create a Separate Build Directory

GCC must always be built in a directory separate from its source tree:

```bash
mkdir build-gcc
cd build-gcc
```

### 5.3 Configure GCC (Stage 1)

```bash
../configure \
    --prefix=$LFSLITE_TOOLS \
    --target=$LFSLITE_TARGET \
    --host=$LFSLITE_HOST \
    --build=$LFSLITE_HOST \
    --with-sysroot=$LFSLITE \
    --with-native-system-header-dir=/usr/include \
    --enable-languages=c,c++ \
    --disable-multilib \
    --disable-bootstrap \
    --disable-nls \
    --disable-libatomic \
    --disable-libgomp \
    --disable-libquadmath \
    --disable-libsanitizer \
    --disable-libssp \
    --disable-libvtv \
    --disable-shared \
    --with-newlib \
    --without-headers \
    CFLAGS="-Os -pipe" \
    CXXFLAGS="-Os -pipe"
```

**Flag explanation:**

| Flag | Purpose |
|------|---------|
| `--prefix=$LFSLITE_TOOLS` | Install GCC into our tools directory |
| `--target=$LFSLITE_TARGET` | Build GCC to produce code for `x86_64-linux-musl` |
| `--with-sysroot=$LFSLITE` | Tell GCC where the target system root is |
| `--with-native-system-header-dir=/usr/include` | Where to find headers inside the sysroot |
| `--enable-languages=c,c++` | Build only C and C++ compilers |
| `--disable-multilib` | Do not build 32-bit libraries alongside 64-bit |
| `--disable-bootstrap` | Skip the 3-stage self-compilation (saves time for stage 1) |
| `--disable-nls` | No localization |
| `--disable-lib*` | Disable optional runtime libraries not needed yet |
| `--disable-shared` | Build only static versions of GCC's runtime libs |
| `--with-newlib` | Indicate there is no real libc yet (stage 1 only) |
| `--without-headers` | Do not use system headers during the GCC build itself |

### 5.4 Build and Install (Stage 1)

```bash
make $MAKEFLAGS all-gcc all-target-libgcc
make install-gcc install-target-libgcc
```

We only build `all-gcc` and `all-target-libgcc` at this stage — the full
`libstdc++` requires a working C library, which we do not have yet.

### 5.5 Create Convenience Symlinks

```bash
ln -sfv $LFSLITE_TOOLS/bin/${LFSLITE_TARGET}-gcc    $LFSLITE_TOOLS/bin/gcc
ln -sfv $LFSLITE_TOOLS/bin/${LFSLITE_TARGET}-g++    $LFSLITE_TOOLS/bin/g++
ln -sfv $LFSLITE_TOOLS/bin/${LFSLITE_TARGET}-ar     $LFSLITE_TOOLS/bin/ar
ln -sfv $LFSLITE_TOOLS/bin/${LFSLITE_TARGET}-ld     $LFSLITE_TOOLS/bin/ld
ln -sfv $LFSLITE_TOOLS/bin/${LFSLITE_TARGET}-ranlib $LFSLITE_TOOLS/bin/ranlib
ln -sfv $LFSLITE_TOOLS/bin/${LFSLITE_TARGET}-strip  $LFSLITE_TOOLS/bin/cross-strip
```

### 5.6 Verify Stage 1

```bash
$LFSLITE_TOOLS/bin/gcc --version
# Should show: gcc (GCC) 15.2.0 ...

echo 'int main(){return 0;}' > /tmp/test-gcc.c
$LFSLITE_TOOLS/bin/gcc -static /tmp/test-gcc.c -o /tmp/test-gcc
file /tmp/test-gcc
# Should show: ELF 64-bit LSB executable, x86-64, statically linked
```

---

## 6. Building Musl (C Library)

Musl is a lightweight, correct, and standards-conformant C library. We use it instead
of glibc because it compiles quickly, links statically without issues, and produces
smaller binaries. The Linux API headers we installed in Section 4 are consumed here.

### 6.1 Extract and Configure

```bash
cd $LFSLITE_SRC
tar xf musl-1.2.6.tar.gz
cd musl-1.2.6

./configure \
    --prefix=$LFSLITE/usr \
    --syslibdir=$LFSLITE/lib \
    --disable-shared \
    --enable-static \
    CC="$LFSLITE_TOOLS/bin/gcc" \
    CFLAGS="-Os -pipe" \
    LIBCC="$LFSLITE_TOOLS/lib/gcc/${LFSLITE_TARGET}/15.2.0/libgcc.a"
```

**Flag explanation:**

| Flag | Purpose |
|------|---------|
| `--prefix=$LFSLITE/usr` | Install headers and libs into our sysroot |
| `--syslibdir=$LFSLITE/lib` | Place the dynamic linker stub in `/lib` |
| `--disable-shared` | Do not build the shared library |
| `--enable-static` | Build the static archive (`libc.a`) |
| `CC=...` | Use our newly built GCC cross-compiler |
| `LIBCC=...` | Point to `libgcc.a` we built in Stage 1 |

### 6.2 Build and Install

```bash
make $MAKEFLAGS
make install
```

### 6.3 Set Up the musl-gcc Wrapper

musl's install creates a `musl-gcc` wrapper script that invokes our toolchain GCC
with all include and library paths redirected into the sysroot:

```bash
ls -la $LFSLITE/usr/bin/musl-gcc

# Make it available on PATH
ln -sfv $LFSLITE/usr/bin/musl-gcc $LFSLITE_TOOLS/bin/musl-gcc
```

Set `CC` to the wrapper for all subsequent package builds:

```bash
export CC="musl-gcc"
export CXX="$LFSLITE_TOOLS/bin/g++"
```

### 6.4 GCC Stage 2 — Full Rebuild Against musl

Now that musl is installed and its headers are in `$LFSLITE/usr/include`, we rebuild
GCC from scratch with a proper configure. Stage 1 was configured with `--with-newlib`,
`--without-headers`, and `--disable-shared` — flags that told GCC to pretend there was
no libc. Those flags must be removed for Stage 2, otherwise GCC will continue to
generate code that makes incorrect assumptions about the runtime environment.

Stage 2 is a completely separate build directory and a fresh `../configure` invocation.
We do **not** reuse the Stage 1 build tree.

```bash
cd $LFSLITE_SRC/gcc-15.2.0
mkdir build-gcc-stage2
cd build-gcc-stage2
```

Configure Stage 2 — notice the absence of `--with-newlib`, `--without-headers`,
and `--disable-shared` compared to Stage 1:

```bash
../configure \
    --prefix=$LFSLITE_TOOLS \
    --target=$LFSLITE_TARGET \
    --host=$LFSLITE_HOST \
    --build=$LFSLITE_HOST \
    --with-sysroot=$LFSLITE \
    --with-native-system-header-dir=/usr/include \
    --enable-languages=c,c++ \
    --disable-multilib \
    --disable-bootstrap \
    --disable-nls \
    --disable-libatomic \
    --disable-libgomp \
    --disable-libquadmath \
    --disable-libsanitizer \
    --disable-libssp \
    --disable-libvtv \
    --enable-shared \
    --enable-threads=posix \
    CFLAGS="-Os -pipe" \
    CXXFLAGS="-Os -pipe"
```

**What changed from Stage 1:**

| Flag | Stage 1 | Stage 2 | Reason |
|------|---------|---------|--------|
| `--with-newlib` | present | **removed** | musl is now the libc; `--with-newlib` would override it |
| `--without-headers` | present | **removed** | musl headers are now installed and should be used |
| `--disable-shared` | present | **removed** → `--enable-shared` | We want shared libraries in the final toolchain |
| `--enable-threads=posix` | absent | **added** | musl supports POSIX threads; enables `libgcc_s` thread support |

Build and install the full Stage 2 GCC:

```bash
make $MAKEFLAGS
make install
```

This replaces the Stage 1 binaries in `$LFSLITE_TOOLS` with the complete compiler.
After this step, `$LFSLITE_TOOLS/bin/gcc` is a fully functional, musl-aware GCC 15.2.0
with C, C++, libgcc, and libstdc++ support.

Update the `musl-gcc` wrapper symlink to point at the freshly installed Stage 2 binary:

```bash
ln -sfv $LFSLITE/usr/bin/musl-gcc $LFSLITE_TOOLS/bin/musl-gcc
```

### 6.5 Verify Stage 2

```bash
# Verify the compiler version
$LFSLITE_TOOLS/bin/gcc --version
# Should show: gcc (GCC) 15.2.0 ...

# Verify libstdc++ is now present
ls $LFSLITE_TOOLS/${LFSLITE_TARGET}/lib/libstdc++*
# Should show: libstdc++.a  libstdc++.so  libstdc++.so.6  ...

# Compile a C++ test against musl
echo '#include <iostream>
int main() { std::cout << "Stage 2 GCC + musl works" << std::endl; return 0; }' \
    > /tmp/test-stage2.cpp

$LFSLITE_TOOLS/bin/g++ -static-libstdc++ -static-libgcc \
    /tmp/test-stage2.cpp -o /tmp/test-stage2
/tmp/test-stage2
# Expected output: Stage 2 GCC + musl works

file /tmp/test-stage2
# Should show: ELF 64-bit LSB executable, x86-64, statically linked
```

### 6.6 Verify musl

```bash
echo '#include <stdio.h>
int main() { puts("musl + gcc work"); return 0; }' > /tmp/test-musl.c

musl-gcc -static /tmp/test-musl.c -o /tmp/test-musl
/tmp/test-musl
# Expected output: musl + gcc work

file /tmp/test-musl
# Should show: ELF 64-bit LSB executable, x86-64, statically linked
```

---

## 7. Building Bash

Bash serves two roles in LFS-Lite: it is the interactive shell, and it will be
used as PID 1 (init) — the first process the kernel hands control to after boot.

### 7.1 Extract and Configure

```bash
cd $LFSLITE_SRC
tar xf bash-5.3.tar.gz
cd bash-5.3

./configure \
    --prefix=/usr \
    --without-bash-malloc \
    --disable-nls \
    --enable-static-link \
    --without-curses \
    CC="musl-gcc" \
    CFLAGS="-Os -pipe" \
    LDFLAGS="-static"
```

**Flag explanation:**

| Flag | Purpose |
|------|---------|
| `--prefix=/usr` | Bash will live at `/usr/bin/bash` inside the new system |
| `--without-bash-malloc` | Use libc malloc instead of Bash's custom allocator (required for musl) |
| `--disable-nls` | Disable internationalization — saves space, no locale files needed |
| `--enable-static-link` | Produce a fully static binary |
| `--without-curses` | Do not link against ncurses (we are not building it) |
| `LDFLAGS="-static"` | Force static linking at link time |

### 7.2 Build and Install

```bash
make $MAKEFLAGS
make DESTDIR=$LFSLITE install
```

### 7.3 Create Essential Symlinks

```bash
mkdir -pv $LFSLITE/bin
ln -sfv /usr/bin/bash $LFSLITE/bin/bash
ln -sfv /usr/bin/bash $LFSLITE/bin/sh
ln -sfv /usr/bin/bash $LFSLITE/usr/bin/sh
```

### 7.4 Verify

```bash
file $LFSLITE/usr/bin/bash
# Should show: ELF 64-bit LSB executable, statically linked

$LFSLITE/usr/bin/bash --version | head -1
# Should show: GNU bash, version 5.3...
```

---

## 8. Building Coreutils

GNU Coreutils provides the fundamental file, text, and shell utilities — `ls`, `cp`,
`mv`, `cat`, `mkdir`, `chmod`, and many others. Without these, a shell has nothing
to work with.

### 8.1 Extract and Configure

```bash
cd $LFSLITE_SRC
tar xf coreutils-9.10.tar.xz
cd coreutils-9.10

./configure \
    --prefix=/usr \
    --enable-install-program=hostname \
    --disable-nls \
    --disable-acl \
    --disable-xattr \
    --without-openssl \
    --without-gmp \
    CC="musl-gcc" \
    CFLAGS="-Os -pipe" \
    LDFLAGS="-static"
```

**Flag explanation:**

| Flag | Purpose |
|------|---------|
| `--enable-install-program=hostname` | Also install the `hostname` utility |
| `--disable-nls` | No localization |
| `--disable-acl` | No POSIX ACL support (no libacl in our sysroot) |
| `--disable-xattr` | No extended attribute support |
| `--without-openssl` | Do not use OpenSSL for hashing utilities |
| `--without-gmp` | Do not use GMP for arbitrary precision in `factor` |

### 8.2 Build and Install

```bash
make $MAKEFLAGS
make DESTDIR=$LFSLITE install
```

### 8.3 Create Essential Symlinks

```bash
mkdir -pv $LFSLITE/bin $LFSLITE/sbin
for tool in cat cp ls mkdir mv rm ln chmod chown echo; do
    ln -sfv /usr/bin/$tool $LFSLITE/bin/$tool
done
```

### 8.4 Verify

```bash
$LFSLITE/usr/bin/ls --version | head -1
# Should output: ls (GNU coreutils) 9.10
```

---

## 9. Building Util-Linux

Util-Linux provides essential system-level utilities: `mount`, `umount`, `fdisk`,
`blkid`, `lsblk`, `swapon`, `losetup`, and many more. These are indispensable for
any real system.

### 9.1 Extract and Configure

```bash
cd $LFSLITE_SRC
tar xf util-linux-2.42.tar.xz
cd util-linux-2.42

./configure \
    --prefix=/usr \
    --disable-nls \
    --disable-tls \
    --without-python \
    --without-systemd \
    --without-udev \
    --disable-makeinstall-chown \
    --disable-makeinstall-setuid \
    --disable-liblastlog2 \
    --disable-libblkid-unicode \
    --without-readline \
    --without-ncurses \
    --without-ncursesw \
    --without-tinfo \
    CC="musl-gcc" \
    CFLAGS="-Os -pipe" \
    LDFLAGS="-static"
```

**Flag explanation:**

| Flag | Purpose |
|------|---------|
| `--without-python` | No Python bindings |
| `--without-systemd` | No systemd integration |
| `--without-udev` | No udev integration |
| `--disable-makeinstall-chown` | Do not attempt to chown files during install |
| `--disable-makeinstall-setuid` | Do not setuid any binaries during install |
| `--disable-liblastlog2` | Disable lastlog2 (requires a separate library) |
| `--without-readline` | No readline for interactive tools |
| `--without-ncurses` | No ncurses (we are not building it) |

### 9.2 Build and Install

```bash
make $MAKEFLAGS
make DESTDIR=$LFSLITE install
```

---

## 10. Building the Linux Kernel

The kernel is the heart of the system. We will build a minimal but functional
x86-64 kernel with just enough drivers to boot in a VM (virtio, VGA console)
and on common real hardware (SATA/AHCI, USB).

> We already extracted the Linux 7.0 sources in Section 4 for the headers step.
> We reuse the same source tree here.

### 10.1 Enter the Source Tree

```bash
cd $LFSLITE_SRC/linux-7.0
```

### 10.2 Start from a Minimal Configuration

Rather than starting from `defconfig` (which enables hundreds of unnecessary options),
we start from `allnoconfig` and enable only what we need:

```bash
make mrproper
make allnoconfig
make menuconfig
```

### 10.3 Required Kernel Options

In `menuconfig`, navigate to and enable the following options.
Options marked `[*]` are built-in (preferred — avoids module complexity).

**64-bit kernel:**
```
General setup --->
    [*] 64-bit kernel
```

**Executable formats:**
```
Executable file formats --->
    [*] Kernel support for ELF binaries
    [*] Kernel support for scripts starting with #!
```

**Virtual filesystems:**
```
File systems --->
    [*] /proc file system support
    [*] sysfs file system support
    [*] Tmpfs virtual memory file system support
    [*] Devtmpfs file system support
    [*]   Automount devtmpfs at /dev after the kernel mounts the root
```

**Block devices and storage:**
```
Device Drivers --->
    [*] Block devices
    [*] SCSI device support --->
        [*] SCSI disk support
    [*] Serial ATA and Parallel ATA drivers (libata) --->
        [*] AHCI SATA support
    [*] Virtio block driver
```

**PCI and USB:**
```
Bus support --->
    [*] PCI support

Device Drivers --->
    [*] USB support --->
        [*] xHCI HCD (USB 3.0) support
        [*] EHCI HCD (USB 2.0) support
        [*] USB Mass Storage support
```

**Console and display:**
```
Device Drivers --->
    Character devices --->
        [*] Enable TTY
    Graphics support --->
        [*] VGA text console
        [*] Framebuffer Console support
```

**ISO 9660 filesystem (to read the ISO at boot):**
```
File systems --->
    CD-ROM/DVD Filesystems --->
        [*] ISO 9660 CDROM file system support
        [*]   Microsoft Joliet CDROM extensions
```

**Initramfs support:**
```
General setup --->
    [*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
```

**EFI stub (for UEFI boot):**
```
Processor type and features --->
    [*] EFI runtime service support
    [*]   EFI stub support
```

Save the configuration and exit menuconfig.

### 10.4 Build the Kernel

The kernel must be compiled with a plain GCC — we use our toolchain GCC directly,
not the `musl-gcc` wrapper. The kernel is a freestanding binary and does not link
against any libc:

```bash
make ARCH=x86_64 \
     CC="$LFSLITE_TOOLS/bin/${LFSLITE_TARGET}-gcc" \
     $MAKEFLAGS \
     bzImage
```

The output kernel image will be at `arch/x86_64/boot/bzImage`.

Copy it into our sysroot for later use:

```bash
mkdir -pv $LFSLITE/boot
cp -v arch/x86_64/boot/bzImage $LFSLITE/boot/vmlinuz
cp -v .config $LFSLITE/boot/config-7.0
```

---

## 11. Building GRUB

GRUB is our bootloader. We will build it with support for both BIOS (i386-pc) and
EFI (x86_64-efi) targets so the ISO is universally bootable.

GRUB requires the **host** GCC (not our cross-compiler) and must be configured
twice — once per target platform.

### 11.1 Extract

```bash
cd $LFSLITE_SRC
tar xf grub-2.14.tar.xz
cd grub-2.14
```

### 11.2 Build for BIOS (i386-pc)

```bash
mkdir build-bios && cd build-bios

../configure \
    --prefix=$LFSLITE_TOOLS \
    --target=i386 \
    --with-platform=pc \
    --disable-efiemu \
    --disable-nls \
    --disable-werror \
    CC=gcc \
    TARGET_CC=gcc \
    CFLAGS="-Os"

make $MAKEFLAGS
make install
cd ..
```

### 11.3 Build for EFI (x86_64-efi)

```bash
mkdir build-efi && cd build-efi

../configure \
    --prefix=$LFSLITE_TOOLS \
    --target=x86_64 \
    --with-platform=efi \
    --disable-nls \
    --disable-werror \
    CC=gcc \
    TARGET_CC=gcc \
    CFLAGS="-Os"

make $MAKEFLAGS
make install
cd ..
```

**Flag explanation:**

| Flag | Purpose |
|------|---------|
| `--target=i386 --with-platform=pc` | Build GRUB for BIOS/MBR boot |
| `--target=x86_64 --with-platform=efi` | Build GRUB for UEFI boot |
| `--disable-efiemu` | Skip the EFI emulator (not needed) |
| `--disable-nls` | No localization |
| `--disable-werror` | Do not treat warnings as errors (avoids failures on newer host GCC) |

---

## 12. Assembling the Root Filesystem

Now we assemble a complete, minimal root filesystem that will be packed into an
initramfs. The kernel will extract this into RAM at boot and execute `/init` —
which we point to Bash.

### 12.1 Final Sysroot Layout

At this point, `$LFSLITE` should contain:

```
$LFSLITE/
├── bin/        → symlinks to /usr/bin tools
├── boot/
│   ├── vmlinuz
│   └── config-7.0
├── dev/        (empty — kernel populates this)
├── etc/
├── lib/        → musl dynamic linker stub
├── lib64/
├── proc/       (empty — kernel mounts procfs here)
├── sys/        (empty — kernel mounts sysfs here)
├── run/
├── root/
├── tmp/
└── usr/
    ├── bin/    → bash, ls, cp, cat, mount, ...
    ├── include/ → Linux headers + musl headers
    └── lib/    → libc.a, libgcc.a, libstdc++.a, ...
```

### 12.2 Create `/init`

The kernel's first userspace process is `/init`. We write a shell script that
mounts essential filesystems and drops into a Bash shell:

```bash
cat > $LFSLITE/init << 'EOF'
#!/bin/bash
# LFS-Lite /init — PID 1

# Mount essential virtual filesystems
mount -t proc  none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev 2>/dev/null || mount -t tmpfs none /dev

# Create basic device nodes if devtmpfs is unavailable
[ -c /dev/console ] || mknod /dev/console c 5 1
[ -c /dev/null    ] || mknod /dev/null    c 1 3
[ -c /dev/tty     ] || mknod /dev/tty     c 5 0

# Print banner
echo ""
echo "  ██╗     ███████╗███████╗      ██╗     ██╗████████╗███████╗"
echo "  ██║     ██╔════╝██╔════╝      ██║     ██║╚══██╔══╝██╔════╝"
echo "  ██║     █████╗  ███████╗█████╗██║     ██║   ██║   █████╗  "
echo "  ██║     ██╔══╝  ╚════██║╚════╝██║     ██║   ██║   ██╔══╝  "
echo "  ███████╗██║     ███████║      ███████╗██║   ██║   ███████╗ "
echo "  ╚══════╝╚═╝     ╚══════╝      ╚══════╝╚═╝   ╚═╝   ╚══════╝ "
echo ""
echo "  Welcome to LFS-Lite"
echo "  Kernel : $(uname -r)"
echo "  GCC    : 15.2.0"
echo "  musl   : 1.2.6"
echo ""

# Drop into interactive shell
export PS1="[lfs-lite]# "
export PATH=/usr/bin:/bin:/usr/sbin:/sbin
exec /bin/bash
EOF

chmod +x $LFSLITE/init
```

### 12.3 Create Minimal `/etc` Files

Some tools expect these files to exist, even minimally:

```bash
cat > $LFSLITE/etc/passwd << 'EOF'
root:x:0:0:root:/root:/bin/bash
EOF

cat > $LFSLITE/etc/group << 'EOF'
root:x:0:
EOF

cat > $LFSLITE/etc/hostname << 'EOF'
lfs-lite
EOF
```

### 12.4 Strip Binaries

Strip debug symbols to reduce the initramfs size. Use our cross-strip so it
understands the ELF format we produced:

```bash
find $LFSLITE/usr/bin $LFSLITE/bin -type f \
    -exec $LFSLITE_TOOLS/bin/cross-strip --strip-unneeded {} \; 2>/dev/null
```

---

## 13. Creating the Bootable ISO

We pack the sysroot into a CPIO initramfs, then wrap it with the kernel and GRUB
into a hybrid BIOS/EFI ISO image.

### 13.1 Create the initramfs

```bash
cd $LFSLITE

find . | cpio -oH newc | gzip -9 > /tmp/lfs-lite-initramfs.cpio.gz

ls -lh /tmp/lfs-lite-initramfs.cpio.gz
# A stripped build should be well under 50 MB
```

### 13.2 Set Up the ISO Staging Tree

```bash
ISO_ROOT=/tmp/lfs-lite-iso
mkdir -pv $ISO_ROOT/{boot/grub,EFI/BOOT}

cp -v $LFSLITE/boot/vmlinuz               $ISO_ROOT/boot/vmlinuz
cp -v /tmp/lfs-lite-initramfs.cpio.gz     $ISO_ROOT/boot/initramfs.cpio.gz
```

### 13.3 Write the GRUB Configuration

```bash
cat > $ISO_ROOT/boot/grub/grub.cfg << 'EOF'
set default=0
set timeout=3

menuentry "LFS-Lite" {
    linux  /boot/vmlinuz quiet
    initrd /boot/initramfs.cpio.gz
}

menuentry "LFS-Lite (verbose boot)" {
    linux  /boot/vmlinuz
    initrd /boot/initramfs.cpio.gz
}
EOF
```

### 13.4 Create the EFI Boot Image

For UEFI support, we embed a FAT-formatted EFI System Partition image inside the ISO:

```bash
# Build the GRUB EFI binary
$LFSLITE_TOOLS/bin/grub-mkimage \
    --format=x86_64-efi \
    --output=$ISO_ROOT/EFI/BOOT/BOOTX64.EFI \
    --prefix=/boot/grub \
    part_gpt part_msdos fat iso9660 normal boot linux echo \
    all_video gfxterm gfxmenu ls search search_label

# Create a 4 MB FAT image to hold the EFI binary
dd if=/dev/zero of=$ISO_ROOT/boot/efi.img bs=1M count=4
mkfs.fat -F 12 $ISO_ROOT/boot/efi.img

# Copy the EFI binary into the FAT image
mmd   -i $ISO_ROOT/boot/efi.img ::/EFI
mmd   -i $ISO_ROOT/boot/efi.img ::/EFI/BOOT
mcopy -i $ISO_ROOT/boot/efi.img $ISO_ROOT/EFI/BOOT/BOOTX64.EFI ::/EFI/BOOT/
```

### 13.5 Build the BIOS GRUB Core Image

```bash
$LFSLITE_TOOLS/bin/grub-mkimage \
    --format=i386-pc \
    --output=/tmp/grub-core.img \
    --prefix=/boot/grub \
    biosdisk part_gpt part_msdos iso9660 normal boot linux echo \
    all_video gfxterm ls search search_label

# Prepend the BIOS CD boot sector
cat $LFSLITE_TOOLS/lib/grub/i386-pc/cdboot.img /tmp/grub-core.img \
    > $ISO_ROOT/boot/grub/bios.img

# Copy runtime GRUB modules into the ISO tree
cp -r $LFSLITE_TOOLS/lib/grub/i386-pc    $ISO_ROOT/boot/grub/
cp -r $LFSLITE_TOOLS/lib/grub/x86_64-efi $ISO_ROOT/boot/grub/
```

### 13.6 Assemble the Final ISO

```bash
xorriso -as mkisofs \
    -iso-level 3 \
    -full-iso9660-filenames \
    -volid "LFS-LITE" \
    -eltorito-boot boot/grub/bios.img \
        -no-emul-boot \
        -boot-load-size 4 \
        -boot-info-table \
        --grub2-boot-info \
        --grub2-mbr $LFSLITE_TOOLS/lib/grub/i386-pc/boot_hybrid.img \
    -eltorito-alt-boot \
        -e boot/efi.img \
        -no-emul-boot \
        -isohybrid-gpt-basdat \
    -append_partition 2 C12A7328-F81F-11D2-BA4B-00A0C93EC93B $ISO_ROOT/boot/efi.img \
    -output /tmp/lfs-lite.iso \
    $ISO_ROOT

ls -lh /tmp/lfs-lite.iso
```

Your ISO is ready at `/tmp/lfs-lite.iso`.

---

## 14. Testing in QEMU

Before writing to a USB drive, test your ISO in QEMU.

### 14.1 BIOS Boot Test

```bash
qemu-system-x86_64 \
    -m 512M \
    -cdrom /tmp/lfs-lite.iso \
    -boot d \
    -nographic \
    -serial stdio
```

### 14.2 UEFI Boot Test

You will need an OVMF firmware image for EFI emulation:

```bash
# Debian/Ubuntu:
sudo apt install ovmf
OVMF=/usr/share/OVMF/OVMF_CODE.fd

# Arch Linux:
# sudo pacman -S edk2-ovmf
# OVMF=/usr/share/OVMF/x64/OVMF_CODE.fd

# Fedora:
# sudo dnf install edk2-ovmf
# OVMF=/usr/share/OVMF/OVMF_CODE.fd

qemu-system-x86_64 \
    -m 512M \
    -drive if=pflash,format=raw,readonly=on,file=$OVMF \
    -cdrom /tmp/lfs-lite.iso \
    -boot d \
    -nographic \
    -serial stdio
```

### 14.3 Expected Boot Sequence

1. GRUB menu appears with "LFS-Lite" and "LFS-Lite (verbose boot)" entries
2. Kernel decompresses and initializes hardware
3. Kernel mounts the initramfs
4. Kernel executes `/init`
5. `/init` mounts `/proc`, `/sys`, `/dev`
6. LFS-Lite banner is displayed (kernel version, GCC version, musl version)
7. A Bash prompt appears: `[lfs-lite]#`

### 14.4 Sanity Checks at the Prompt

Once booted, run these to verify everything works:

```bash
uname -r              # Should show: 7.0
ls /proc              # procfs mounted
ls /dev               # devtmpfs mounted
ls /usr/bin | head    # Coreutils present
mount                 # Show mounted filesystems
hostname              # Should print: lfs-lite
```

### 14.5 Writing to a USB Drive

If you want to boot on real hardware:

```bash
# DANGER: Replace /dev/sdX with your actual USB device.
# This will ERASE the USB drive completely.
sudo dd if=/tmp/lfs-lite.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

The ISO is hybrid — bootable as a CD-ROM and as a USB drive in both BIOS and UEFI modes.

---

## 15. Package Versions Reference

| Package     | Version | Source URL |
|-------------|---------|------------|
| Linux       | 7.0     | https://cdn.kernel.org/pub/linux/kernel/v7.x/ |
| GCC         | 15.2.0  | https://ftp.gnu.org/gnu/gcc/gcc-15.2.0/ |
| musl        | 1.2.6   | https://musl.libc.org/releases/ |
| Bash        | 5.3     | https://ftp.gnu.org/gnu/bash/ |
| Coreutils   | 9.10    | https://ftp.gnu.org/gnu/coreutils/ |
| Util-Linux  | 2.42    | https://www.kernel.org/pub/linux/utils/util-linux/v2.42/ |
| GRUB        | 2.14    | https://ftp.gnu.org/gnu/grub/ |

---

## Appendix A: Troubleshooting

### Kernel panics with "No working init found"

The kernel could not execute `/init`. Common causes:

- `/init` is not executable — run `chmod +x $LFSLITE/init`
- `/init` references `/bin/bash` which does not exist — check that the symlinks
  in Section 7.3 were created correctly
- The initramfs was not built from inside `$LFSLITE` — the `find . | cpio` command
  must be run from within `$LFSLITE` itself, not from the parent directory

### GRUB drops to rescue shell

The `grub.cfg` path is wrong or modules are missing. At the rescue prompt:

```
grub> ls
grub> ls (cd0)/boot/grub/
```

Make sure `grub.cfg` exists and the module directories were copied in Section 13.5.

### GCC configure fails: "cannot compute suffix of object files"

Your host GCC is not functional, or GMP/MPFR/MPC headers are missing. Install the
`-dev` / `-devel` packages for all three and retry from a clean build directory.

### Bash says "not found" for basic commands

The `PATH` inside `/init` may be wrong, or Coreutils binaries were installed to a
different prefix. Verify:

```bash
ls $LFSLITE/usr/bin/
ls $LFSLITE/bin/
```

### Build fails: "musl-gcc: command not found"

Ensure `$LFSLITE_TOOLS/bin` is on your PATH:

```bash
export PATH=$LFSLITE_TOOLS/bin:$PATH
which musl-gcc
```

### `strip` produces "file format not recognized"

You may be using the host `strip` on cross-compiled ELF files. Always use the
cross-strip symlink we created:

```bash
$LFSLITE_TOOLS/bin/cross-strip --strip-unneeded <binary>
```

### musl configure fails: "libgcc.a not found"

The path in `LIBCC=` must match your GCC version exactly. Verify:

```bash
find $LFSLITE_TOOLS/lib/gcc -name "libgcc.a"
# Use the path returned here in your musl configure LIBCC= argument
```

---

## Appendix B: Going Further

LFS-Lite is intentionally minimal. Once you have a working ISO, natural next steps are:

- **Add a real init system** — replace `/init` with OpenRC or runit
- **Add more userland** — busybox, findutils, grep, sed, awk
- **Add a package manager** — apk-tools or xbps
- **Persistent storage** — mount a real partition instead of booting purely into RAM
- **Networking** — build iproute2 and a DHCP client (dhcpcd)
- **Display** — add a framebuffer console font or even a minimal X server
- **Full LFS** — https://www.linuxfromscratch.org covers all of these and much more

---

*LFS-Lite — Build small. Learn everything.*
