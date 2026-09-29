# Embedded Linux ARM64 Simulation Lab: U-Boot, Kernel, BusyBox Initramfs & Kernel Modules

An end-to-end, production-grade guide for building, cross-compiling, and simulating an **ARM64 Embedded Linux System** from source code using **QEMU**, the **Linux Kernel**, **BusyBox**, **U-Boot**, and **Out-of-Tree Kernel Modules**.

---

## Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Host Environment & Prerequisites](#2-host-environment--prerequisites)
3. [Building & Testing U-Boot Bootloader](#3-building--testing-u-boot-bootloader)
4. [Cross-Compiling the Linux Kernel (ARM64)](#4-cross-compiling-the-linux-kernel-arm64)
5. [Building BusyBox & Creating Initramfs RootFS](#5-building-busybox--creating-initramfs-rootfs)
6. [Writing & Compiling an Out-of-Tree Kernel Module](#6-writing--compiling-an-out-of-tree-kernel-module)
7. [Packaging the Final `initramfs.cpio.gz`](#7-packaging-the-final-initramfscpiogz)
8. [Full System Simulation in QEMU](#8-full-system-simulation-in-qemu)
9. [Module Verification & Runtime Testing](#9-module-verification--runtime-testing)
10. [Troubleshooting & Common Pitfalls](#10-troubleshooting--common-pitfalls)

---

## 1. Architecture Overview

```
+-------------------------------------------------------------+
|                      Hardware Layer                         |
|      QEMU ARM64 Virt Platform (-M virt -cpu cortex-a76)      |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|                     Bootloader Layer                        |
|                  U-Boot (u-boot.bin)                        |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|                       Kernel Layer                          |
|             Linux Kernel ARM64 (arch/arm64/boot/Image)      |
+-------------------------------------------------------------+
                              |
       +----------------------+----------------------+
       |                                             |
       v                                             v
+-----------------------------+       +------------------------------+
|     Root Filesystem         |       |      Dynamic Extensions      |
|  BusyBox Static Initramfs   | <---> |     Linux Kernel Module      |
|    (initramfs.cpio.gz)      |       |      (test-module.ko)        |
+-----------------------------+       +------------------------------+
```

---

## 2. Host Environment & Prerequisites

This workflow requires a POSIX-compatible 64-bit Linux build environment:
* **Recommended:** Ubuntu 22.04 / 24.04 LTS (via native Linux, WSL2 on Windows, or GitHub Codespaces).
* **Note for Windows users:** Native PowerShell cannot compile U-Boot or the Linux Kernel due to case-sensitivity, POSIX script requirements, and symlink handling. Use **WSL2** (`wsl --install`) or an **Ubuntu Cloud Instance**.

### 2.1 Install Build Dependencies
Execute the following commands on your Linux host:

```bash
sudo apt-get update && sudo apt-get install -y \
    build-essential \
    bison \
    flex \
    bc \
    libssl-dev \
    device-tree-compiler \
    swig \
    python3-dev \
    python3-pyelftools \
    cpio \
    qemu-system-arm \
    gcc \
    gcc-aarch64-linux-gnu \
    git \
    wget \
    tar \
    gzip
```

### 2.2 Verify the Cross-Compiler
```bash
aarch64-linux-gnu-gcc --version
```
*Expected: `aarch64-linux-gnu-gcc (Ubuntu ...) 13.x.x` or similar.*

---

## 3. Building & Testing U-Boot Bootloader

### 3.1 Clone the U-Boot Repository
```bash
cd /workspaces/codespaces-blank
git clone --depth 1 https://source.denx.de/u-boot/u-boot.git
cd u-boot
```

### 3.2 Configure and Compile for ARM64 Virtual Machine
```bash
# Configure for QEMU ARM64 target
make qemu_arm64_defconfig

# Cross-compile using all CPU cores
CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)
```

Confirm that the output binary exists:
```bash
ls -lh u-boot.bin
```

### 3.3 Test U-Boot in QEMU
```bash
qemu-system-aarch64 -M virt -cpu cortex-a53 -m 512M -bios u-boot.bin -nographic
```
*To exit QEMU: Press `Ctrl + A`, release, then press `X`.*

---

## 4. Cross-Compiling the Linux Kernel (ARM64)

### 4.1 Clone the Linux Kernel Source
```bash
cd /workspaces/codespaces-blank
git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git -b linux-6.1.y
cd linux
```

### 4.2 Configure and Compile the Kernel
```bash
# 1. Generate default ARM64 configuration
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig

# 2. Compile the uncompressed ARM64 kernel Image
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc) Image

# 3. Generate module layout headers required for out-of-tree modules
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make modules_prepare
```

Verify the kernel binary:
```bash
ls -lh arch/arm64/boot/Image
```

---

## 5. Building BusyBox & Creating Initramfs RootFS

### 5.1 Download and Unpack BusyBox
```bash
cd /workspaces/codespaces-blank
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xf busybox-1.36.1.tar.bz2
cd busybox-1.36.1
```

### 5.2 Configure BusyBox (Static Build & Patching)
BusyBox must be compiled statically so it runs independently of external glibc runtime dependencies in the initial RAM filesystem:

```bash
# Generate baseline configuration
make defconfig

# Enable static binary compilation
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config

# Disable 'tc' to prevent build failure on Linux 6.8+ kernel headers
sed -i 's/CONFIG_TC=y/# CONFIG_TC is not set/' .config
sed -i 's/CONFIG_FEATURE_TC_INGRESS=y/# CONFIG_FEATURE_TC_INGRESS is not set/' .config
```

### 5.3 Cross-Compile and Install BusyBox
```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make install
```

Verify that BusyBox was built for **ARM64** and is **statically linked**:
```bash
file ./_install/bin/busybox
```
*Output must contain: `ELF 64-bit LSB executable, ARM aarch64, ..., statically linked`.*

### 5.4 Build the Root Filesystem Hierarchy
```bash
mkdir -p initramfs
cd initramfs

# Copy all BusyBox symlinks and binaries
cp -a ../_install/* .

# Create essential virtual filesystem mountpoints
mkdir -p dev proc sys etc root usr/modules
```

### 5.5 Write the `/init` Initialization Script
The kernel runs `/init` as **PID 1**. This script mounts the virtual filesystems and launches the interactive shell:

```bash
cat << 'EOF' > init
#!/bin/sh
mount -t devtmpfs devtmpfs /dev
mount -t proc none /proc
mount -t sysfs none /sys
echo "========================================="
echo " Welcome to Minimal Embedded Linux!      "
echo "========================================="
exec /bin/sh
EOF

# Grant execution rights
chmod +x init
```

---

## 6. Writing & Compiling an Out-of-Tree Kernel Module

### 6.1 Create Module Directory and Source Code
```bash
mkdir -p /workspaces/codespaces-blank/kernel-module
cd /workspaces/codespaces-blank/kernel-module

cat << 'EOF' > test-module.c
#include <linux/init.h>
#include <linux/module.h>

static int __init test_init(void)
{
    printk(KERN_INFO "Hello World from Kernel Module!\n");
    return 0;
}

static void __exit test_exit(void)
{
    printk(KERN_INFO "Goodbye World from Kernel Module!\n");
}

module_init(test_init);
module_exit(test_exit);

MODULE_AUTHOR("Embedded Linux Developer");
MODULE_DESCRIPTION("Test Hello World Kernel Module");
MODULE_LICENSE("GPL");
EOF
```

### 6.2 Create the Kbuild Makefile
*(Note: Recipes must be preceded by a real Tab character).*

```bash
printf 'obj-m += test-module.o\nKDIR ?= /workspaces/codespaces-blank/linux\n\nall:\n\t$(MAKE) -C $(KDIR) M=$(PWD) modules\n\nclean:\n\t$(MAKE) -C $(KDIR) M=$(PWD) clean\n' > Makefile
```

### 6.3 Cross-Compile the Kernel Module
```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make
```

Verify that `test-module.ko` is created:
```bash
ls -lh test-module.ko
file test-module.ko
```

---

## 7. Packaging the Final `initramfs.cpio.gz`

1. **Copy the compiled kernel module into the initramfs structure:**
   ```bash
   cp /workspaces/codespaces-blank/kernel-module/test-module.ko \
      /workspaces/codespaces-blank/busybox-1.36.1/initramfs/usr/modules/
   ```

2. **Package and compress using `cpio` and `gzip`:**
   ```bash
   cd /workspaces/codespaces-blank/busybox-1.36.1/initramfs
   find . -print0 | cpio --null -ov --format=newc | gzip -9 > /workspaces/codespaces-blank/initramfs.cpio.gz
   ```

3. **Verify the generated image:**
   ```bash
   ls -lh /workspaces/codespaces-blank/initramfs.cpio.gz
   ```
   *Expected size: approximately 1.2 MB – 2.5 MB.*

---

## 8. Full System Simulation in QEMU

Launch the complete stack (Kernel + Initramfs) using QEMU's ARM64 virtual machine target:

```bash
cd /workspaces/codespaces-blank/linux

qemu-system-aarch64 -M virt -cpu cortex-a76 -nographic -smp 1 \
    -kernel ./arch/arm64/boot/Image \
    -append "console=ttyAMA0" \
    -m 2048 \
    -initrd /workspaces/codespaces-blank/initramfs.cpio.gz
```

---

## 9. Module Verification & Runtime Testing

Once the system boots, you will be greeted by the root prompt (`/ #` or `~ #`):

### 9.1 Verify Operating Environment
```sh
uname -a
cat /proc/cpuinfo
ls -l /usr/modules/
```

### 9.2 Insert the Kernel Module
```sh
insmod /usr/modules/test-module.ko
```
*Expected kernel message:*
```text
[    x.xxxxxx] test_module: loading out-of-tree module taints kernel.
[    x.xxxxxx] Hello World from Kernel Module!
```

### 9.3 Inspect Active Modules in Kernel Memory
```sh
lsmod
```
*Output will display `test_module` along with its memory size and reference count.*

### 9.4 Remove the Kernel Module
```sh
rmmod test_module
```
*Expected kernel message:*
```text
[    x.xxxxxx] Goodbye World from Kernel Module!
```

Confirm removal:
```sh
lsmod
# Returns empty
```

### 9.5 Shutdown the Virtual Machine
```sh
poweroff -f
```
*(Or press `Ctrl + A`, release, then press `X`).*

---

## 10. Troubleshooting & Common Pitfalls

### 1. `Failed to execute /init (error -8)` or `Kernel panic - not syncing: No working init found`
* **Root Cause:** Error `-8` is `ENOEXEC` (Exec format error). BusyBox was built using the host x86_64 compiler instead of the cross-compiler, or dynamic linking was used without bundling `ld-linux` and shared libc libraries.
* **Fix:** Re-run BusyBox compilation with `ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-`, ensuring `CONFIG_STATIC=y` is active.

### 2. `networking/tc.c: error: 'TCA_CBQ_MAX' undeclared`
* **Root Cause:** Linux Kernel 6.8+ deprecated and removed the CBQ scheduler headers. Older BusyBox builds fail to compile `tc.c`.
* **Fix:** Disable `CONFIG_TC` in BusyBox's `.config` using `sed`:
  ```bash
  sed -i 's/CONFIG_TC=y/# CONFIG_TC is not set/' .config
  sed -i 's/CONFIG_FEATURE_TC_INGRESS=y/# CONFIG_FEATURE_TC_INGRESS is not set/' .config
  ```

### 3. `make[2]: *** No rule to make target 'scripts/module.lds', needed by 'test-module.ko'`
* **Root Cause:** Building out-of-tree kernel modules requires internal kernel linker scripts that are not built by default with `make Image`.
* **Fix:** Run module preparation inside the Linux kernel source directory:
  ```bash
  cd /workspaces/codespaces-blank/linux
  ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make modules_prepare
  ```

### 4. `qemu-system-aarch64: could not load initrd ...`
* **Root Cause:** The `initramfs.cpio.gz` archive does not exist at the specified path or `cpio` failed during archive creation.
* **Fix:** Ensure `cpio` is installed (`sudo apt install -y cpio`), verify your current directory, and use an absolute path when running `find . -print0 | cpio ...`.

### 5. `Your display is too small to run Menuconfig!`
* **Root Cause:** `ncurses` requires at least 19 rows by 80 columns.
* **Fix:** Enlarge your terminal pane or bypass the interactive menu by executing `make defconfig`.
