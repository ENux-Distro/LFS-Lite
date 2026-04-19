# LFS-Lite
### Build a Minimal Linux System and Boot It as an ISO Image

---

> **LFS-Lite** is a condensed, practical guide inspired by Linux From Scratch.
> Instead of building a full POSIX userland, you will build only the six essential
> components needed for a real, bootable system — compiled entirely from source.
> The result is a hybrid Legacy/EFI ISO image you can run in a VM or burn to a USB drive.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Setting Up the Build Environment](#3-setting-up-the-build-environment)
4. [Building Musl (C Library)](#4-building-musl-c-library)
5. [Building Bash](#5-building-bash)
6. [Building Coreutils](#6-building-coreutils)
7. [Building Util-Linux](#7-building-util-linux)
8. [Building the Linux Kernel](#8-building-the-linux-kernel)
9. [Building GRUB](#9-building-grub)
10. [Assembling the Root Filesystem](#10-assembling-the-root-filesystem)
11. [Creating the Bootable ISO](#11-creating-the-bootable-iso)
12. [Testing in QEMU](#12-testing-in-qemu)
13. [Package Versions Reference](#13-package-versions-reference)

---

## 1. Introduction

### What Is LFS-Lite?

Linux From Scratch (LFS) is a legendary project that teaches you to build a complete
Linux system from source code. It is thorough, educational, and takes days to complete.

LFS-Lite is for people who understand *why* LFS exists but want a shorter path to a
real, working result. You will build exactly six components:

| Component   | Role                                      |
|-------------|-------------------------------------------|
| musl        | The C standard library                    |
| Bash        | The shell / PID 1 init                    |
| Coreutils   | Core POSIX utilities (ls, cp, cat, …)     |
| Util-Linux  | Essential system tools (mount, fdisk, …)  |
| Linux       | The kernel                                |
| GRUB        | The bootloader (BIOS + EFI)               |

The output is a bootable ISO image. Everything is compiled from source on your host
machine using a cross-compilation-like workflow, then assembled into an initramfs and
a GRUB-bootable ISO.

### What You Will Learn

- How each component fits together in a Linux system
- How to pass `./configure` flags for a minimal, musl-based build
- How to build a Linux kernel from scratch
- How to create an initramfs and wrap it into a bootable ISO with GRUB

### What This Guide Is Not

- A full LFS replacement — you will not get a package manager, init system, or a
  rich userland
- A cross-compilation tutorial — we compile natively on the host but install into
  a separate sysroot
- A production system — this is for learning

---

## 2. Prerequisites

### Host Tools Required

Your host system (any Linux distro) must have the following tools available.
Install them through your package manager before starting.

```
gcc           g++           make          bison
flex          gawk          texinfo       help2man
pkg-config    libssl-dev    xorriso       grub-pc-bin
grub-efi-amd64-bin          mtools        libelf-dev
libncurses-dev              python3
```

**Debian/Ubuntu:**
```bash
sudo apt install build-essential bison flex gawk texinfo help2man \
    pkg-config libssl-dev xorriso grub-pc-bin grub-efi-amd64-bin \
    mtools libelf-dev libncurses-dev python3
```

**Arch Linux:**
```bash
sudo pacman -S base-devel bison flex gawk texinfo help2man \
    pkg-config openssl xorriso grub mtools libelf ncurses python
```

**Fedora/RHEL:**
```bash
sudo dnf install gcc gcc-c++ make bison flex gawk texinfo help2man \
    pkgconfig openssl-devel xorriso grub2-tools mtools elfutils-libelf-devel \
    ncurses-devel python3
```

### Disk Space

You will need approximately **4 GB** of free disk space:
- ~2 GB for extracted source trees and build artifacts
- ~500 MB for the final sysroot
- ~200 MB for the ISO image

### Architecture

This guide targets **x86-64 (amd64)**. The resulting ISO will be hybrid BIOS/EFI bootable.

---

## 3. Setting Up the Build Environment

### 3.1 Directory Structure

We will use a consistent directory layout throughout this guide. All work happens
inside `$LFS`, which acts as our sysroot — the root of the new system.

```bash
export LFS=/opt/lfs-lite
export LFS_SRC=$LFS/sources
export LFS_TOOLS=$LFS/tools

mkdir -pv $LFS/{bin,lib,lib64,usr/{bin,lib,include},etc,dev,proc,sys,run,tmp}
mkdir -pv $LFS_SRC
mkdir -pv $LFS_TOOLS
```

Add `$LFS` to your shell environment permanently for this session:

```bash
# Add to your ~/.bashrc or run in every new terminal before building
export LFS=/opt/lfs-lite
export LFS_SRC=$LFS/sources
export LFS_TOOLS=$LFS/tools
export PATH=$LFS_TOOLS/bin:$PATH
```

### 3.2 Downloading Sources

Download all source tarballs into `$LFS_SRC`. See [Section 13](#13-package-versions-reference)
for the exact versions used in this guide.

```bash
cd $LFS_SRC

wget https://musl.libc.org/releases/musl-1.2.5.tar.gz
wget https://ftp.gnu.org/gnu/bash/bash-5.2.tar.gz
wget https://ftp.gnu.org/gnu/coreutils/coreutils-9.5.tar.xz
wget https://www.kernel.org/pub/linux/utils/util-linux/v2.40/util-linux-2.40.tar.xz
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.9.tar.xz
wget https://ftp.gnu.org/gnu/grub/grub-2.12.tar.xz
```

Verify the downloads completed:

```bash
ls -lh $LFS_SRC
```

### 3.3 A Note on Build Flags

Throughout this guide we will use a consistent set of compiler flags. These optimize
the output for a small, self-contained system:

```bash
export CFLAGS="-Os -pipe"
export CXXFLAGS="-Os -pipe"
export MAKEFLAGS="-j$(nproc)"
```

`-Os` optimizes for size rather than speed — appropriate for a minimal system.
`-j$(nproc)` parallelizes the build across all available CPU cores.

---

## 4. Building Musl (C Library)

Musl is a lightweight, correct, and standards-conformant C library. We use it instead
of glibc because it compiles quickly, links statically without issues, and produces
smaller binaries. Everything else in this guide will be linked against musl.

### 4.1 Extract and Configure

```bash
cd $LFS_SRC
tar xf musl-1.2.5.tar.gz
cd musl-1.2.5

./configure \
    --prefix=$LFS/usr \
    --syslibdir=$LFS/lib \
    --disable-shared \
    --enable-static \
    CFLAGS="-Os -pipe"
```

**Flag explanation:**

| Flag | Purpose |
|------|---------|
| `--prefix=$LFS/usr` | Install headers and libs into our sysroot |
| `--syslibdir=$LFS/lib` | Place `libc.so` (the dynamic linker) in `/lib` |
| `--disable-shared` | Do not build the shared library |
| `--enable-static` | Build the static archive (`libc.a`) |

### 4.2 Build and Install

```bash
make $MAKEFLAGS
make install
```

### 4.3 Create the musl-gcc Wrapper

musl ships a wrapper script called `musl-gcc` that invokes your host GCC but redirects
all includes and library paths into our sysroot. We will use this wrapper when building
all subsequent packages.

```bash
# The musl install already created $LFS/usr/bin/musl-gcc
# Verify it exists:
ls -la $LFS/usr/bin/musl-gcc

# Make it available on PATH
ln -sfv $LFS/usr/bin/musl-gcc $LFS_TOOLS/bin/musl-gcc
```

Set the compiler for all subsequent builds:

```bash
export CC="musl-gcc"
export CXX="g++"   # C++ is only used by GRUB; keep host g++ for that
```

### 4.4 Verify

```bash
echo '#include <stdio.h>
int main() { puts("musl works"); return 0; }' > /tmp/test.c

musl-gcc -static /tmp/test.c -o /tmp/test-musl
/tmp/test-musl
# Expected output: musl works

file /tmp/test-musl
# Should show: statically linked
```

---

## 5. Building Bash

Bash serves two roles in LFS-Lite: it is the interactive shell, and it will be
used as PID 1 (init) — the first process the kernel hands control to after boot.

### 5.1 Extract and Configure

```bash
cd $LFS_SRC
tar xf bash-5.2.tar.gz
cd bash-5.2

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

### 5.2 Build and Install

```bash
make $MAKEFLAGS
make DESTDIR=$LFS install
```

### 5.3 Create the `/bin/sh` Symlink

```bash
ln -sfv /usr/bin/bash $LFS/usr/bin/sh
mkdir -pv $LFS/bin
ln -sfv /usr/bin/bash $LFS/bin/bash
ln -sfv /usr/bin/bash $LFS/bin/sh
```

### 5.4 Verify

```bash
file $LFS/usr/bin/bash
# Should show: statically linked
```

---

## 6. Building Coreutils

GNU Coreutils provides the fundamental file, text, and shell utilities — `ls`, `cp`,
`mv`, `cat`, `mkdir`, `chmod`, and many others. Without these, a shell has nothing
to work with.

### 6.1 Extract and Configure

```bash
cd $LFS_SRC
tar xf coreutils-9.5.tar.xz
cd coreutils-9.5

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
| `--disable-acl` | No POSIX ACL support (no libacl available) |
| `--disable-xattr` | No extended attribute support |
| `--without-openssl` | Do not use OpenSSL for hashing utilities |
| `--without-gmp` | Do not use GMP for arbitrary precision in `factor` |

### 6.2 Build and Install

```bash
make $MAKEFLAGS
make DESTDIR=$LFS install
```

### 6.3 Create Essential Symlinks

Older scripts and the kernel's init path expect certain tools under `/bin`:

```bash
mkdir -pv $LFS/bin $LFS/sbin
for tool in cat cp ls mkdir mv rm ln chmod chown; do
    ln -sfv /usr/bin/$tool $LFS/bin/$tool
done
```

### 6.4 Verify

```bash
$LFS/usr/bin/ls --version | head -1
# Should output: ls (GNU coreutils) 9.5
```

---

## 7. Building Util-Linux

Util-Linux provides essential system-level utilities: `mount`, `umount`, `fdisk`,
`blkid`, `lsblk`, `swapon`, `losetup`, and many more. These are indispensable for
any real system.

### 7.1 Extract and Configure

```bash
cd $LFS_SRC
tar xf util-linux-2.40.tar.xz
cd util-linux-2.40

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
| `--disable-makeinstall-chown` | Do not attempt to chown files during install (avoids root requirement) |
| `--disable-makeinstall-setuid` | Do not setuid any binaries during install |
| `--disable-liblastlog2` | Disable lastlog2 (requires separate library) |
| `--without-readline` | No readline for interactive tools |
| `--without-ncurses` | No ncurses (we are not building it) |

### 7.2 Build and Install

```bash
make $MAKEFLAGS
make DESTDIR=$LFS install
```

---

## 8. Building the Linux Kernel

The kernel is the heart of the system. We will build a minimal but functional
x86-64 kernel with just enough drivers to boot in a VM (virtio, VGA console)
and on common real hardware (SATA/AHCI, USB).

### 8.1 Extract Sources

```bash
cd $LFS_SRC
tar xf linux-6.9.tar.xz
cd linux-6.9
```

### 8.2 Start from a Minimal Configuration

Rather than starting from `defconfig` (which enables hundreds of unnecessary options),
we start from `allnoconfig` and enable only what we need:

```bash
make mrproper        # Clean any stale config or build artifacts
make allnoconfig     # Start with everything disabled
make menuconfig      # Open the interactive configuration menu
```

### 8.3 Required Kernel Options

In `menuconfig`, navigate to and enable the following options.
Options marked `[*]` are built-in; `[M]` would be a module (avoid modules for simplicity).

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

**Process filesystem (required for many tools):**
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
    [*] Virtio block driver   (for QEMU)
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

**Console and framebuffer:**
```
Device Drivers --->
    Character devices --->
        [*] Enable TTY
    Graphics support --->
        [*] VGA text console
        [*] Framebuffer Console support
```

**ISO 9660 filesystem (needed to read the ISO at boot):**
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

### 8.4 Build the Kernel

```bash
# Use the host gcc for the kernel, not musl-gcc
make ARCH=x86_64 CC=gcc $MAKEFLAGS bzImage
```

> **Note:** The kernel must be compiled with the host GCC, not `musl-gcc`.
> The kernel is a freestanding binary and does not link against any libc.

The output kernel image will be at:
```
arch/x86_64/boot/bzImage
```

Copy it into our sysroot for later use:

```bash
mkdir -pv $LFS/boot
cp -v arch/x86_64/boot/bzImage $LFS/boot/vmlinuz
cp -v .config $LFS/boot/config-6.9
```

---

## 9. Building GRUB

GRUB is our bootloader. We will build it with support for both BIOS (i386-pc) and
EFI (x86_64-efi) targets so the ISO is universally bootable.

### 9.1 Extract and Configure

GRUB requires the host GCC (not musl-gcc) and must be configured twice — once per
target platform.

```bash
cd $LFS_SRC
tar xf grub-2.12.tar.xz
cd grub-2.12
```

**Build for BIOS (i386-pc):**

```bash
mkdir build-bios && cd build-bios

../configure \
    --prefix=$LFS_TOOLS \
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

**Build for EFI (x86_64-efi):**

```bash
mkdir build-efi && cd build-efi

../configure \
    --prefix=$LFS_TOOLS \
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
| `--disable-werror` | Do not treat warnings as errors (avoids build failures on newer GCC) |

---

## 10. Assembling the Root Filesystem

Now we assemble a complete, minimal root filesystem that will be packed into an
initramfs (initial RAM filesystem). The kernel will extract this into RAM at boot
and execute `/init` — which we will point to Bash.

### 10.1 Final Sysroot Layout

At this point, `$LFS` should contain:

```
$LFS/
├── bin/        → symlinks to /usr/bin tools
├── boot/
│   └── vmlinuz
├── dev/        (empty — kernel populates this)
├── etc/
├── lib/        → musl dynamic linker (libc.so)
├── lib64/      → symlink or duplicate
├── proc/       (empty — kernel mounts procfs here)
├── sys/        (empty — kernel mounts sysfs here)
├── run/
├── tmp/
└── usr/
    ├── bin/    → bash, ls, cp, cat, mount, ...
    ├── include/
    └── lib/    → libc.a, libmusl, ...
```

### 10.2 Create `/init`

The kernel's first userspace process is `/init`. We write a simple shell script
that mounts essential filesystems and drops into a Bash shell:

```bash
cat > $LFS/init << 'EOF'
#!/bin/bash
# LFS-Lite /init — PID 1

# Mount essential virtual filesystems
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev 2>/dev/null || mount -t tmpfs none /dev

# Create basic device nodes if devtmpfs failed
[ -c /dev/console ] || mknod /dev/console c 5 1
[ -c /dev/null ]    || mknod /dev/null    c 1 3
[ -c /dev/tty ]     || mknod /dev/tty     c 5 0

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
echo "  Kernel: $(uname -r)"
echo ""

# Drop into interactive shell
export PS1="[lfs-lite]# "
export PATH=/usr/bin:/bin:/usr/sbin:/sbin
exec /bin/bash
EOF

chmod +x $LFS/init
```

### 10.3 Create `/etc/passwd` and `/etc/group`

Some tools expect these files to exist, even minimally:

```bash
cat > $LFS/etc/passwd << 'EOF'
root:x:0:0:root:/root:/bin/bash
EOF

cat > $LFS/etc/group << 'EOF'
root:x:0:
EOF

cat > $LFS/etc/hostname << 'EOF'
lfs-lite
EOF
```

### 10.4 Strip Binaries

Strip debug symbols to minimize the initramfs size:

```bash
find $LFS/usr/bin $LFS/bin -type f -exec strip --strip-unneeded {} \; 2>/dev/null
```

---

## 11. Creating the Bootable ISO

We will now pack the sysroot into a CPIO initramfs, then wrap it with the kernel
and GRUB into a hybrid BIOS/EFI ISO image.

### 11.1 Create the initramfs

```bash
cd $LFS

find . | cpio -oH newc | gzip -9 > /tmp/lfs-lite-initramfs.cpio.gz

ls -lh /tmp/lfs-lite-initramfs.cpio.gz
```

The initramfs should be well under 50 MB for a stripped build.

### 11.2 Set Up the ISO Tree

Create a staging directory for the ISO contents:

```bash
ISO_ROOT=/tmp/lfs-lite-iso
mkdir -pv $ISO_ROOT/{boot/grub,EFI/BOOT}

# Copy kernel and initramfs
cp -v $LFS/boot/vmlinuz        $ISO_ROOT/boot/vmlinuz
cp -v /tmp/lfs-lite-initramfs.cpio.gz  $ISO_ROOT/boot/initramfs.cpio.gz
```

### 11.3 Write the GRUB Configuration

```bash
cat > $ISO_ROOT/boot/grub/grub.cfg << 'EOF'
set default=0
set timeout=3

menuentry "LFS-Lite" {
    linux  /boot/vmlinuz quiet
    initrd /boot/initramfs.cpio.gz
}

menuentry "LFS-Lite (verbose)" {
    linux  /boot/vmlinuz
    initrd /boot/initramfs.cpio.gz
}
EOF
```

### 11.4 Create the EFI Boot Image

For UEFI support, we need a FAT-formatted EFI System Partition image embedded in the ISO:

```bash
# Build the GRUB EFI binary
$LFS_TOOLS/bin/grub-mkimage \
    --format=x86_64-efi \
    --output=$ISO_ROOT/EFI/BOOT/BOOTX64.EFI \
    --prefix=/boot/grub \
    part_gpt part_msdos fat iso9660 normal boot linux echo \
    all_video gfxterm gfxmenu ls search search_label

# Create a small FAT image to hold the EFI binary
dd if=/dev/zero of=$ISO_ROOT/boot/efi.img bs=1M count=4
mkfs.fat -F 12 $ISO_ROOT/boot/efi.img

# Copy the EFI binary into the FAT image
mmd     -i $ISO_ROOT/boot/efi.img ::/EFI
mmd     -i $ISO_ROOT/boot/efi.img ::/EFI/BOOT
mcopy   -i $ISO_ROOT/boot/efi.img $ISO_ROOT/EFI/BOOT/BOOTX64.EFI ::/EFI/BOOT/
```

### 11.5 Build the BIOS GRUB Core Image

```bash
$LFS_TOOLS/bin/grub-mkimage \
    --format=i386-pc \
    --output=/tmp/grub-core.img \
    --prefix=/boot/grub \
    biosdisk part_gpt part_msdos iso9660 normal boot linux echo \
    all_video gfxterm ls search search_label

# Combine with the BIOS boot sector
cat $LFS_TOOLS/lib/grub/i386-pc/cdboot.img /tmp/grub-core.img \
    > $ISO_ROOT/boot/grub/bios.img
```

Copy the GRUB modules needed at runtime:

```bash
cp -r $LFS_TOOLS/lib/grub/i386-pc    $ISO_ROOT/boot/grub/
cp -r $LFS_TOOLS/lib/grub/x86_64-efi $ISO_ROOT/boot/grub/
```

### 11.6 Assemble the Final ISO

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
        --grub2-mbr $LFS_TOOLS/lib/grub/i386-pc/boot_hybrid.img \
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

## 12. Testing in QEMU

Before writing to a USB drive, test your ISO in QEMU.

### 12.1 BIOS Boot Test

```bash
qemu-system-x86_64 \
    -m 512M \
    -cdrom /tmp/lfs-lite.iso \
    -boot d \
    -nographic \
    -serial stdio
```

### 12.2 UEFI Boot Test

You will need an OVMF firmware image for EFI emulation:

```bash
# Debian/Ubuntu:
sudo apt install ovmf
OVMF=/usr/share/OVMF/OVMF_CODE.fd

# Arch Linux:
# sudo pacman -S edk2-ovmf
# OVMF=/usr/share/OVMF/x64/OVMF_CODE.fd

qemu-system-x86_64 \
    -m 512M \
    -bios $OVMF \
    -cdrom /tmp/lfs-lite.iso \
    -boot d \
    -nographic \
    -serial stdio
```

### 12.3 Expected Boot Sequence

1. GRUB menu appears with "LFS-Lite" entry
2. Kernel decompresses and initializes hardware
3. Kernel mounts the initramfs
4. Kernel executes `/init`
5. `/init` mounts `/proc`, `/sys`, `/dev`
6. LFS-Lite banner is displayed
7. A Bash prompt appears: `[lfs-lite]#`

### 12.4 Sanity Checks at the Prompt

Once booted, run these to verify everything works:

```bash
uname -r              # Kernel version
ls /proc              # procfs mounted
ls /dev               # devtmpfs mounted
ls /usr/bin | head    # Coreutils installed
mount | head          # Mounted filesystems
hostname              # Should print "lfs-lite"
```

### 12.5 Writing to a USB Drive

If you want to boot on real hardware:

```bash
# DANGER: Replace /dev/sdX with your actual USB device.
# This will ERASE the USB drive completely.
sudo dd if=/tmp/lfs-lite.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

The ISO is hybrid — it is bootable both as a CD-ROM and as a USB drive in both
BIOS and UEFI modes.

---

## 13. Package Versions Reference

This table lists the exact versions used when this guide was written. Newer versions
will generally work, but may require minor adjustments to configure flags.

| Package     | Version | Source URL |
|-------------|---------|------------|
| musl        | 1.2.5   | https://musl.libc.org/releases/ |
| Bash        | 5.2     | https://ftp.gnu.org/gnu/bash/ |
| Coreutils   | 9.5     | https://ftp.gnu.org/gnu/coreutils/ |
| Util-Linux  | 2.40    | https://www.kernel.org/pub/linux/utils/util-linux/ |
| Linux       | 6.9     | https://www.kernel.org/pub/linux/kernel/v6.x/ |
| GRUB        | 2.12    | https://ftp.gnu.org/gnu/grub/ |

---

## Appendix A: Troubleshooting

### Kernel panics with "No working init found"

The kernel could not execute `/init`. Common causes:

- `/init` is not executable — run `chmod +x $LFS/init`
- `/init` references `/bin/bash` which does not exist in the sysroot — check symlinks
- The initramfs was not built from inside `$LFS` — the `find . | cpio` command must
  be run from within `$LFS`

### GRUB drops to rescue shell

The `grub.cfg` path is wrong or the modules are missing. Boot into the rescue shell
and run:

```
grub> ls
grub> ls (cd0)/boot/grub/
```

Make sure `grub.cfg` exists at the path GRUB expects.

### Bash says "not found" for basic commands

Your `PATH` inside `/init` may be wrong, or the Coreutils binaries were installed
to a different prefix. Run `ls /usr/bin` to check what is present.

### Build fails: "musl-gcc: command not found"

Make sure `$LFS_TOOLS/bin` is on your PATH:

```bash
export PATH=$LFS_TOOLS/bin:$PATH
which musl-gcc
```

---

## Appendix B: Going Further

LFS-Lite is intentionally minimal. Once you have a working ISO, natural next steps are:

- **Add a real init system** — replace `/init` with OpenRC or runit
- **Add more userland** — busybox, findutils, grep, sed, awk
- **Add a package manager** — apk-tools or xbps
- **Persistent storage** — mount a real partition instead of booting purely into RAM
- **Networking** — build iproute2 and a DHCP client
- **Display** — add a framebuffer console or even a minimal X server

The full Linux From Scratch book at https://www.linuxfromscratch.org covers all of
these in detail if you want to go deeper.

---

*LFS-Lite — Build small. Learn everything.*
