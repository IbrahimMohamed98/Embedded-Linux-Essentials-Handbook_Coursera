# Embedded Linux ARM64 Simulation Lab: U-Boot, Kernel, BusyBox Initramfs & Kernel Modules

An end-to-end, production-grade guide for building, cross-compiling, and simulating an **ARM64 Embedded Linux System** from source code using **QEMU**, the **Linux Kernel**, **BusyBox**, **U-Boot**, and **Out-of-Tree Kernel Modules**.
## Table of Contents

> **Navigation:** Every Table of Contents entry links to a heading within this Markdown file.

- [1. 1. Linux Kernel Theory: Kernel Architecture Types](#1-linux-kernel-theory-kernel-architecture-types)
- [2. 2. Linux Kernel Space vs User Space](#2-linux-kernel-space-vs-user-space)
- [3. 3. Kernel Modules: Theory Before the Practical Build](#3-kernel-modules-theory-before-the-practical-build)
- [4. 4. Kernel Build Configuration Theory](#4-kernel-build-configuration-theory)
- [5. 5. Device Tree Theory](#5-device-tree-theory)
- [6. 6. U-Boot Theory](#6-u-boot-theory)
- [7. 7. Boot Arguments Theory](#7-boot-arguments-theory)
- [8. 8. Initramfs Theory](#8-initramfs-theory)
- [9. 9. BusyBox Theory](#9-busybox-theory)
- [10. 10. `insmod`, `lsmod`, and `rmmod`](#10-insmod-lsmod-and-rmmod)
- [11. 11. `prepare` vs `modules_prepare`](#11-prepare-vs-modules-prepare)
- [12. 12. Why External Modules Must Use Kbuild](#12-why-external-modules-must-use-kbuild)
- [13. 13. Theory-to-Practice Map](#13-theory-to-practice-map)
- [14. 14. Core Concepts & Embedded Linux Boot Architecture](#14-core-concepts-embedded-linux-boot-architecture)
- [15. 15. Host vs. Target Architecture & Cross-Compilation](#15-host-vs-target-architecture-cross-compilation)
- [16. 16. Toolchain and Host Environment Setup](#16-toolchain-and-host-environment-setup)
- [17. 17. Building and Testing the U-Boot Bootloader](#17-building-and-testing-the-u-boot-bootloader)
- [18. 18. Cross-Compiling the Linux Kernel (ARM64)](#18-cross-compiling-the-linux-kernel-arm64)
- [19. 19. Constructing the User Space with BusyBox (Initramfs)](#19-constructing-the-user-space-with-busybox-initramfs)
- [20. 20. Writing and Compiling an Out-of-Tree Kernel Module](#20-writing-and-compiling-an-out-of-tree-kernel-module)
- [21. 21. Packaging the Initramfs Root Filesystem](#21-packaging-the-initramfs-root-filesystem)
- [22. 22. Booting and Running in QEMU](#22-booting-and-running-in-qemu)
- [23. 23. Runtime Module Verification Inside QEMU](#23-runtime-module-verification-inside-qemu)
- [24. 24. Troubleshooting Post-Mortem: Errors and Technical Lessons](#24-troubleshooting-post-mortem-errors-and-technical-lessons)
- [25. 25. Chat Follow-Up: Rebuilding the Kernel and Preparing for External Modules](#25-chat-follow-up-rebuilding-the-kernel-and-preparing-for-external-modules)
- [26. 26. Compact Mental Model](#26-compact-mental-model)
- [27. 27. Device Drivers, `/dev`, and `/sys`](#27-device-drivers-dev-and-sys)

---

## 1. Linux Kernel Theory: Kernel Architecture Types

Before building Linux, it is important to understand what a kernel actually is and the major ways operating-system kernels can be organized.

### 1.1 What is the kernel?

The **kernel** is the core part of an operating system. It sits between user-space programs and the hardware.

```text
┌─────────────────────────────┐
│         User Space          │
│  Shell / Applications       │
│  BusyBox / Services        │
└──────────────┬──────────────┘
               │ system calls
               ▼
┌─────────────────────────────┐
│           Kernel            │
│ Memory / Processes / Drivers│
│ Filesystems / Networking    │
│ Scheduling / Hardware       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│          Hardware           │
│ CPU / RAM / UART / Storage  │
│ GPIO / Network / Devices    │
└─────────────────────────────┘
```

The kernel provides controlled access to hardware and manages CPU time, memory, devices, filesystems, networking, processes, threads, and interrupts.

---

### 1.2 Microkernel

A **microkernel** keeps the privileged kernel core as small as possible. Many services, such as drivers and filesystems, can run outside the core.

```text
┌──────────────────────────────────┐
│           User Space             │
│ File Server / Drivers / Network  │
└──────────────┬───────────────────┘
               │ IPC
               ▼
┌──────────────────────────────────┐
│          Microkernel             │
│ Scheduling / IPC / Memory        │
└──────────────────────────────────┘
               │
               ▼
            Hardware
```

**Advantages:** strong separation, smaller privileged core, and potential fault isolation.

**Disadvantages:** IPC and service separation can introduce overhead and architectural complexity.

**Key idea:** Microkernel = keep the kernel core small and move many services outside it.

---

### 1.3 Monolithic Kernel

A **monolithic kernel** places many major operating-system services inside kernel space.

```text
┌─────────────────────────────────────┐
│              Kernel Space           │
│ Process / Memory / Drivers          │
│ Filesystems / Networking / Security │
└──────────────────┬──────────────────┘
                   │
                   ▼
                Hardware
```

**Advantages:** high performance and direct communication between kernel subsystems.

**Disadvantages:** a serious fault in kernel-space code can affect the whole system, and the privileged code base is larger.

**Key idea:** Monolithic = many major OS services operate inside kernel space.

---

### 1.4 Modular Kernel

A **modular kernel** can dynamically load and unload parts of its functionality. Linux kernel modules normally use the `.ko` extension.

```text
                 Linux Kernel
              ┌───────────────┐
              │ Kernel Core    │
              └───────┬───────┘
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
          module   module   module
           .ko      .ko      .ko
```

Examples:

```bash
insmod test-module.ko
lsmod
rmmod test_module
```

**Advantages:** functionality can be added when needed, useful during development, and the base kernel can remain smaller.

**Disadvantages:** dependencies and loading/unloading must be managed, and a faulty module still runs in kernel space.

**Key idea:** Modular = kernel functionality can be built and loaded as separate modules.

---

### 1.5 Is Linux monolithic or modular?

Linux is commonly described as a:

> **Monolithic kernel with modular capabilities.**

These terms describe different aspects.

```text
Linux
 │
 ├── Monolithic architecture
 │     └── Major services operate in kernel space
 │
 └── Modular capability
       └── Functionality can also be loaded as .ko modules
```

Therefore, saying that Linux is modular does **not** mean Linux is a microkernel.

---

### 1.6 Kernel architecture comparison

| Property | Microkernel | Monolithic | Modular |
|---|---|---|---|
| Main idea | Minimal privileged core | Many services in kernel | Load functionality dynamically |
| Drivers | Often outside core | Commonly in kernel space | Can be loadable modules |
| IPC importance | High | Lower for internal kernel components | Depends on component |
| Runtime loading | Not defining feature | Not required | Core capability |
| Linux classification | No | Yes | Yes, as a capability |

**Important:** “monolithic” and “modular” are not necessarily opposites. Linux demonstrates this clearly.

---

---


## 2. Linux Kernel Space vs User Space

Normal applications and utilities execute in **user space** and have restricted access to hardware and memory.

The kernel executes in **kernel space** with high privileges.

```text
User Space
────────────────────────────
BusyBox / Shell / Applications
              │
              │ system calls
              ▼
Kernel Space
────────────────────────────
Scheduler / Memory / Drivers
Filesystem / Network / Modules
              │
              ▼
Hardware
```

A module such as:

```text
test-module.ko
```

runs in **kernel space**, not user space.

This is why a buggy kernel module can be much more serious than a normal application crash.

---


## 3. Kernel Modules: Theory Before the Practical Build

A kernel module is compiled separately from the main kernel and can be loaded into the running kernel.

```text
test-module.c
      │
      ▼
Kernel Kbuild
      │
      ▼
test-module.ko
      │
      ▼
insmod
      │
      ▼
Running kernel
```

The `.ko` file is a **loadable kernel object**.

### 3.1 Why use modules?

Instead of rebuilding and rebooting the complete kernel after every change:

```text
Modify source
     ↓
Recompile module
     ↓
Load new .ko
     ↓
Test
```

This makes driver and kernel-feature development much faster.

### 3.2 Module lifecycle

```text
test-module.ko
      │
    insmod
      ▼
Running in kernel
      │
    rmmod
      ▼
Removed
```

---


## 4. Kernel Build Configuration Theory

Linux uses **Kconfig** to manage kernel configuration. The resulting configuration is normally stored in:

```text
.config
```

Typical flow:

```text
Kconfig files
     │
     ▼
defconfig / menuconfig
     │
     ▼
.config
     │
     ▼
Kernel build system
     │
     ▼
Kernel + modules
```

### 4.1 `defconfig`

Creates a baseline configuration:

```bash
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" defconfig
```

### 4.2 `menuconfig`

Provides an interactive interface for selecting features.

### 4.3 `mrproper`

Performs a deep cleanup and can remove `.config`.

Therefore:

```text
make mrproper
      ↓
.config removed
      ↓
configure again
      ↓
defconfig / menuconfig
```

### 4.4 Built-in vs module vs disabled

Kconfig options commonly result in three states:

```text
CONFIG_FEATURE=y
    ↓
Built into the kernel

CONFIG_FEATURE=m
    ↓
Built as a .ko module

# CONFIG_FEATURE is not set
    ↓
Disabled
```

---


## 5. Device Tree Theory

Embedded Linux commonly uses a **Device Tree** to describe hardware to the kernel.

The Device Tree is a description of hardware, not the hardware itself.

```text
Hardware
   │
   │ described by
   ▼
Device Tree
   │
   │ interpreted by
   ▼
Linux kernel
   │
   ▼
Drivers
```

A compiled Device Tree is commonly a:

```text
.dtb
```

It can describe CPUs, memory, UARTs, GPIO controllers, I2C/SPI devices, interrupts, clocks, and other hardware resources.

---


## 6. U-Boot Theory

U-Boot is a bootloader widely used in embedded systems.

Conceptually:

```text
Power On
   │
   ▼
U-Boot
   ├── Initialize basic hardware
   ├── Read boot configuration
   ├── Load kernel
   ├── Load Device Tree
   ├── Set boot arguments
   │
   ▼
Linux kernel
```

Important U-Boot concepts include:

- `bootcmd` — commands used for the normal boot sequence.
- `bootargs` — Linux kernel command-line arguments.
- `U_BOOT_CMD` — mechanism for registering U-Boot commands.
- `U_BOOT_DRIVER` — registers U-Boot drivers.
- `UCLASS` — groups related U-Boot devices/drivers.
- `defconfig` — target-specific configuration.

For QEMU ARM64:

```bash
make qemu_arm64_defconfig
```

selects the U-Boot configuration for that target.

---


## 7. Boot Arguments Theory

Linux receives a command line from the bootloader.

For example:

```text
console=ttyAMA0,115200
root=/dev/mmcblk0p2
rw
rootwait
earlycon
```

Meaning:

```text
console=...
    ↓
Kernel console

root=...
    ↓
Root filesystem

rw
    ↓
Read/write root filesystem

rootwait
    ↓
Wait for root storage

earlycon
    ↓
Early kernel console
```

For the minimal QEMU system:

```bash
-append "console=ttyAMA0"
```

directs kernel console output to the emulated UART.

---


## 8. Initramfs Theory

An **initramfs** is an initial RAM filesystem.

It gives Linux a temporary root filesystem during early boot.

```text
Linux kernel
     │
     ▼
initramfs
     ├── /init
     ├── /bin
     ├── /dev
     ├── /proc
     ├── /sys
     └── /usr/modules
```

For this project:

```text
BusyBox
   +
/init
   +
test-module.ko
   │
   ▼
initramfs/
   │
   ▼
cpio + gzip
   │
   ▼
initramfs.cpio.gz
   │
   ▼
QEMU -initrd
   │
   ▼
RAM root filesystem
```

---


## 9. BusyBox Theory

BusyBox combines many common Unix utilities into one small executable.

Instead of installing separate programs for:

```text
ls
cp
cat
sh
mount
insmod
rmmod
```

BusyBox provides these utilities from one main executable, usually with links for the individual commands.

This makes it particularly useful for small embedded root filesystems and initramfs environments.

For an initramfs, **static linking** is especially useful because required shared libraries may not exist in the initial filesystem.

---


## 10. `insmod`, `lsmod`, and `rmmod`

### 10.1 Load

```bash
insmod /usr/modules/test-module.ko
```

```text
test-module.ko
      │
    insmod
      ▼
Linux kernel
```

### 10.2 List

```bash
lsmod
```

shows currently loaded modules.

### 10.3 Remove

```bash
rmmod test_module
```

removes the module.

The file can be:

```text
test-module.ko
```

while the loaded module can appear as:

```text
test_module
```

because module naming normalizes the hyphen/underscore representation.

---


## 11. `prepare` vs `modules_prepare`

These targets prepare the kernel source/build tree for later build operations.

### 11.1 `make prepare`

Generates kernel files needed by subsequent build stages.

### 11.2 `make modules_prepare`

Prepares the kernel tree for building external modules and generates/builds the module-related infrastructure required by Kbuild.

Typical sequence:

```bash
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" defconfig
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" prepare
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" modules_prepare
```

Then, for a complete kernel:

```bash
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" -j20
```

---


## 12. Why External Modules Must Use Kbuild

Do not normally compile a kernel module with:

```bash
gcc -c test-module.c
```

Instead, use Linux's Kbuild system:

```text
test-module.c
      │
      ▼
Kbuild
      │
      ├── Kernel headers
      ├── .config
      ├── Architecture flags
      ├── Compiler flags
      └── Module linker information
      │
      ▼
test-module.ko
```

The module Makefile contains:

```make
obj-m += test-module.o
```

and invokes the kernel build system with:

```bash
make -C ../linux M=$PWD modules
```

---


## 13. Theory-to-Practice Map

```text
Operating System Theory
          │
   ┌──────┴───────┐
   ▼              ▼
Kernel types   User/Kernel space
   │              │
   ├─ Micro       │
   ├─ Monolithic  │
   └─ Modular     │
          │       │
          └───┬───┘
              ▼
        Linux Kernel
              │
       ┌──────┼────────┐
       ▼      ▼        ▼
    Kconfig Drivers  Modules
       │      │        │
    .config  DTB      .ko
       │      │        │
       └──────┼────────┘
              ▼
           U-Boot
              │
        Kernel + DTB
              │
              ▼
        Linux boot
              │
              ▼
          Initramfs
              │
           BusyBox
              │
              ▼
          User shell
              │
       insmod / lsmod
              │
              ▼
       Kernel module
```

This connects the theory to the practical commands in the later sections.



---
---

## 14. Core Concepts & Embedded Linux Boot Architecture

Unlike general-purpose PC operating systems (which boot via UEFI/BIOS and discovery buses like ACPI and PCIe), an embedded system uses a dedicated, highly controlled boot pipeline:

```
[ ROM / First-Stage Bootloader ]
            │
            ▼
[ Secondary Bootloader (SPL) ]
            │
            ▼
[ Third-Stage Bootloader (U-Boot) ]
            │
            ▼
[ Linux Kernel (Image / zImage) ] + [ Device Tree (DTB) ]
            │
            ▼
[ Initial RAM Filesystem (Initramfs / rootfs) ]
            │
            ▼
[ User Space Process (PID 1: /init or systemd) ]
```

1. **Bootloader (U-Boot):** Initializes basic DRAM, sets up board-level clocks, reads storage (SD, eMMC, flash, or network TFTP), loads the kernel binary and Device Tree into physical memory, and jumps to the kernel execution vector.
2. **Linux Kernel:** Takes control of the MMU, initializes hardware peripherals described by the Device Tree, mounts an initial root filesystem, and launches the first user space process.
3. **Initramfs:** A minimal root filesystem packed as a `cpio` archive and compressed with `gzip`. It resides entirely in RAM, providing user space tools without requiring physical block device drivers to be ready at initial boot.
4. **BusyBox:** Known as the "Swiss Army Knife of Embedded Linux," it combines tiny versions of hundreds of common UNIX utilities (`ls`, `sh`, `mount`, `cp`, `insmod`, `dmesg`) into a single executable binary.

---

## 15. Host vs. Target Architecture & Cross-Compilation

* **Host Machine:** The computer where you write code and compile (x86_64 / Intel/AMD).
* **Target Platform:** The embedded device where the code actually runs (ARM64 / Cortex-A53 / Cortex-A76).

Because an x86_64 CPU cannot execute ARM instructions, we use a **cross-compiler**:
* Standard compiler: `gcc` $\rightarrow$ produces x86_64 machine code for your host.
* Cross-compiler: `aarch64-linux-gnu-gcc` $\rightarrow$ runs on x86_64, but produces 64-bit ARM machine code.

---

## 16. Toolchain and Host Environment Setup

### 16.1 Why Native Windows PowerShell Fails for Kernel Development
Building the Linux kernel and U-Boot requires:
1. **POSIX utilities:** `make`, `sed`, `awk`, `grep`, `bison`, `flex`.
2. **Case-sensitive filesystem:** The Linux kernel contains files whose names differ only by letter case (e.g., `include/uapi/linux/netfilter/xt_DSCP.h` vs `xt_dscp.h`). On standard Windows NTFS, this causes file collisions.
3. **Unix Symbolic Links:** Windows standard user accounts cannot create Linux symlinks without Developer Mode enabled.

*Solution:* Perform all builds inside a real Linux environment (WSL2, an Ubuntu VM, or a cloud instance such as GitHub Codespaces).

### 16.2 Installing Build Dependencies
On Ubuntu 24.04 / 22.04 LTS:

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

#### 16.2.1 What each package does:
* `build-essential`: Installs standard GNU C/C++ compilers, libc headers, and GNU Make.
* `bison` & `flex`: Parser generator and lexical analyzer needed to compile the Linux `Kconfig` and U-Boot configuration parsers.
* `bc`: Arbitrary-precision calculator used by the kernel build scripts to generate timing header files (`timeconst.h`).
* `libssl-dev`: OpenSSL development headers required for cryptographic signing of kernel modules and certificates.
* `device-tree-compiler` (`dtc`): Compiles human-readable Device Tree Source (`.dts`) into binary blobs (`.dtb`).
* `swig` & `python3-pyelftools`: Allows U-Boot and the kernel to parse ELF object files and bind C libraries with Python tools.
* `cpio`: Archives the root filesystem structure into a contiguous byte stream.
* `qemu-system-arm`: Provides the QEMU machine emulator for ARM32 and ARM64 targets (`qemu-system-aarch64`).
* `gcc-aarch64-linux-gnu`: The GNU cross-toolchain targeting 64-bit ARM Linux.

---

## 17. Building and Testing the U-Boot Bootloader

### 17.1 Clone U-Boot
```bash
cd /workspaces/codespaces-blank
git clone --depth 1 https://source.denx.de/u-boot/u-boot.git
cd u-boot
```

### 17.2 Configure U-Boot for QEMU ARM64
U-Boot uses the Linux kernel's `Kconfig` architecture. A `defconfig` is a curated list of non-default configuration options tailored to a specific board:

```bash
make qemu_arm64_defconfig
```
This command reads `configs/qemu_arm64_defconfig` and expands it into a full `.config` file containing memory mappings, console UART drivers, and network configurations suitable for QEMU's `virt` platform.

### 17.3 Compile the Bootloader
```bash
CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)
```
* `CROSS_COMPILE=aarch64-linux-gnu-`: Tells the Makefile to use `aarch64-linux-gnu-gcc`, `aarch64-linux-gnu-ld`, and `aarch64-linux-gnu-ar` instead of host tools.
* `-j$(nproc)`: Queries your CPU count using `nproc` and compiles across all cores in parallel.

The compilation produces `u-boot.bin`, which is the raw executable image.

### 17.4 Test Booting U-Boot in QEMU
```bash
qemu-system-aarch64 -M virt -cpu cortex-a53 -m 512M -bios u-boot.bin -nographic
```

#### 17.4.1 Flag Explanations:
* `-M virt`: Selects the standard QEMU virtual target platform (PCIe, GIC interrupt controller, PL011 UART).
* `-cpu cortex-a53`: Emulates an ARM Cortex-A53 64-bit core.
* `-m 512M`: Allocates 512 megabytes of RAM to the virtual machine.
* `-bios u-boot.bin`: Tells QEMU to execute `u-boot.bin` as the initial ROM/firmware entrypoint.
* `-nographic`: Disables graphical window output and redirects the UART serial console straight to your current terminal.

*To exit QEMU in non-graphical mode:* Hold `Ctrl`, press `A`, release both, then press `X`.

---

## 18. Cross-Compiling the Linux Kernel (ARM64)

### 18.1 Clone the Kernel Source Tree
```bash
cd /workspaces/codespaces-blank
git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git -b linux-6.1.y
cd linux
```

### 18.2 Configure the Kernel for ARM64
```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig
```
* `ARCH=arm64`: Selects the target architecture architecture folder (`arch/arm64`).
* `defconfig`: Generates the baseline configuration file `.config` for general-purpose 64-bit ARM systems.

### 18.3 Build the Kernel Image
```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc) Image
```
* Target `Image`: Produces an uncompressed raw binary at `arch/arm64/boot/Image`. Unlike x86 (which builds compressed `bzImage`), ARM64 QEMU boots uncompressed `Image` files directly and efficiently.

### 18.4 Prepare the Kernel for External Modules (Crucial Step)
Before compiling out-of-tree kernel modules, the kernel source must generate its internal symbols, data structure offsets, and module layout scripts:

```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make prepare
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make modules_prepare
```
#### 18.4.1 Why this is required:
* `make prepare` invokes `bc` to calculate timer tick conversions and generates `include/generated/timeconst.h` and `asm-offsets.h`. Without this, compiling external code throws:
  `fatal error: generated/timeconst.h: No such file or directory`
* `make modules_prepare` compiles `scripts/mod/modpost` and builds `scripts/module.lds`, which is the linker script that guarantees kernel modules match the kernel's memory structure.

---

## 19. Constructing the User Space with BusyBox (Initramfs)

### 19.1 Download and Extract BusyBox
```bash
cd /workspaces/codespaces-blank
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xf busybox-1.36.1.tar.bz2
cd busybox-1.36.1
```

### 19.2 Configure BusyBox: The Critical Static Linking Step
By default, BusyBox links dynamically against the host C library (`libc.so`). In an initial RAM filesystem without shared libraries installed, executing a dynamic binary triggers:
`Kernel panic - not syncing: No working init found. (error -8)`

To prevent this, BusyBox must be configured as a **static binary**:

```bash
# 1. Generate default configuration
make defconfig

# 2. Force static linking
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config

# 3. Disable 'tc' to resolve Linux 6.8+ header conflict (CBQ removal)
sed -i 's/CONFIG_TC=y/# CONFIG_TC is not set/' .config
sed -i 's/CONFIG_FEATURE_TC_INGRESS=y/# CONFIG_FEATURE_TC_INGRESS is not set/' .config
```

### 19.3 Cross-Compile BusyBox
```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make install
```
This builds the software and installs a complete mini-root filesystem inside `./_install`.

### 19.4 Verify Architecture and Static Linking
```bash
file ./_install/bin/busybox
```
*Expected Output:*
```text
./_install/bin/busybox: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, for GNU/Linux 3.7.0, stripped
```

---

## 20. Writing and Compiling an Out-of-Tree Kernel Module

Kernel modules allow device drivers and subsystem extensions to be loaded and unloaded at runtime without rebuilding or rebooting the kernel.

### 20.1 Create the Kernel Module Source Code
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

MODULE_AUTHOR("Mohammed Billoo");
MODULE_DESCRIPTION("Hello World kernel module");
MODULE_LICENSE("GPL");
EOF
```

#### 20.1.1 Code Breakdown:
* `#include <linux/init.h>` & `<linux/module.h>`: Core macros and headers for the module interface.
* `__init`: An optimization macro telling the kernel to drop this initialization function from memory once it finishes running.
* `__exit`: Marks the cleanup code invoked only when unloading the module with `rmmod`.
* `printk()`: Writes messages to the kernel ring buffer (viewable via `dmesg` or the active serial console).
* `module_init()` & `module_exit()`: Registers entry and exit symbols with the kernel's module management system.
* `MODULE_LICENSE("GPL")`: Prevents the kernel from being marked as "tainted" by proprietary code and enables access to GPL-only exported kernel functions.

### 20.2 Create the Kbuild Makefile
Kernel modules cannot be compiled using standard `gcc -c`. They must be evaluated by the kernel's internal build system (`Kbuild`):

```bash
printf 'obj-m += test-module.o\nKDIR ?= /workspaces/codespaces-blank/linux\n\nall:\n\t$(MAKE) -C $(KDIR) M=$(PWD) modules\n\nclean:\n\t$(MAKE) -C $(KDIR) M=$(PWD) clean\n' > Makefile
```

#### 20.2.1 How this Makefile operates:
* `obj-m += test-module.o`: Instructs Kbuild to build `test-module.c` into a loadable module object (`test-module.ko`).
* `-C $(KDIR)`: Switches directory to your compiled Linux kernel tree to inherit its compilation flags, architecture defines, and include headers.
* `M=$(PWD)`: Directs the kernel build system back to your local folder to build the module files out-of-tree.

### 20.3 Build the Module
```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make
```

Verify that the `.ko` file is ready:
```bash
ls -lh test-module.ko
```

---

## 21. Packaging the Initramfs Root Filesystem

### 21.1 Create Directory Tree and Move Binaries
```bash
cd /workspaces/codespaces-blank/busybox-1_36_0
mkdir -p initramfs
cd initramfs

# Copy BusyBox binary links into the root directory
cp -a ../_install/* .

# Create essential directory mountpoints
mkdir -p dev proc sys etc root usr/modules
```

### 21.2 Write the `/init` Entrypoint Script
Create the initial user space process that the kernel runs on startup:

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

chmod +x init
```

#### 21.2.1 Explanation of Mounts:
* `/dev` (`devtmpfs`): Device node filesystem automatically populated by the kernel for serial ports, disks, and TTYs.
* `/proc` (`procfs`): Virtual filesystem exposing process table and kernel data (`/proc/cpuinfo`, `/proc/meminfo`).
* `/sys` (`sysfs`): Hierarchical tree exposing kernel objects, buses, drivers, and power parameters.
* `exec /bin/sh`: Replaces the execution context of PID 1 with an interactive shell.

### 21.3 Copy the Kernel Module and Pack the Archive
```bash
# Copy the compiled kernel module into user storage inside initramfs
cp /workspaces/codespaces-blank/kernel-module/test-module.ko usr/modules/

# Package into cpio archive compressed with gzip
find . -print0 | cpio --null -ov --format=newc | gzip -9 > /workspaces/codespaces-blank/initramfs.cpio.gz
```

#### 21.3.1 Command Breakdown:
* `find . -print0`: Lists every file in the directory separated by null bytes (`\0`) to handle special characters safely.
* `cpio --null -ov --format=newc`: Reads null-separated file inputs, lists verbose progress (`v`), creates an archive (`o`), and formats it using modern SVR4 portable format with CRC (`newc`), which the Linux kernel expects.
* `gzip -9`: Compresses the archive with maximum compression level.

---

## 22. Booting and Running in QEMU

With the kernel image and initramfs generated, run QEMU:

```bash
cd /workspaces/codespaces-blank/linux

qemu-system-aarch64 -M virt -cpu cortex-a76 -nographic -smp 1 \
    -kernel ./arch/arm64/boot/Image \
    -append "console=ttyAMA0" \
    -m 2048 \
    -initrd /workspaces/codespaces-blank/initramfs.cpio.gz
```

### 22.1 Detailed Parameter Analysis:
| Flag | Value | Purpose |
| :--- | :--- | :--- |
| **`-M`** | `virt` | Emulates ARM's generic reference board with GIC interrupt controllers and VirtIO buses. |
| **`-cpu`** | `cortex-a76` | Sets modern 64-bit ARMv8.2-A CPU execution profile. |
| **`-nographic`** | *(Flag)* | Disables virtual VGA output and redirects all UART input/output to the current shell. |
| **`-smp`** | `1` | Configures symmetric multiprocessing (allocates 1 virtual CPU core). |
| **`-kernel`** | `./arch/arm64/boot/Image` | Passes the raw compiled Linux kernel binary directly to memory. |
| **`-append`** | `"console=ttyAMA0"` | Kernel command line parameter designating ARM's PL011 UART as the primary system console. |
| **`-m`** | `2048` | Allocates 2 GB of virtual system DRAM. |
| **`-initrd`** | `.../initramfs.cpio.gz` | Loads the initial RAM filesystem into memory and passes its memory pointer to the kernel. |

---

## 23. Runtime Module Verification Inside QEMU

When the boot logs settle, you are dropped into the BusyBox shell:

```text
=========================================
 Welcome to Minimal Embedded Linux!      
=========================================
/ # 
```

### 23.1 Verify System Integrity
```sh
uname -a
# Linux (none) 6.1.93-gfbd8b3facb36 #1 SMP PREEMPT aarch64 GNU/Linux

cat /proc/cpuinfo
# Shows Cortex-A76 processor registers and features

ls -l /usr/modules/
# Shows test-module.ko (approx. 33 KB)
```

### 23.2 Load the Kernel Module
```sh
insmod /usr/modules/test-module.ko
```
*Kernel ring buffer output:*
```text
[   15.421092] test_module: loading out-of-tree module taints kernel.
[   15.424103] Hello World from Kernel Module!
```

### 23.3 Inspect Loaded Modules in RAM
```sh
lsmod
```
*Output:*
```text
Module                  Size  Used by
test_module            16384  0
```

### 23.4 Unload the Kernel Module
```sh
rmmod test_module
```
*Kernel ring buffer output:*
```text
[   22.189540] Goodbye World from Kernel Module!
```

Confirm that the module is completely removed from kernel memory:
```sh
lsmod
# Returns empty
```

### 23.5 Shut Down the Virtual Machine
```sh
poweroff -f
```
*(Or press `Ctrl + A`, release, then press `X`).*

---

## 24. Troubleshooting Post-Mortem: Errors and Technical Lessons

### 24.1 `Failed to execute /init (error -8)` / `Kernel panic - not syncing: No working init found`
* **Root Cause:** Error code `-8` corresponds to `ENOEXEC` (*Exec format error*). This occurs when:
  1. BusyBox was compiled with the host compiler (x86_64) instead of the target cross-compiler (`aarch64-linux-gnu-`). The ARM64 kernel cannot parse x86 machine instructions.
  2. BusyBox was dynamically linked, but the C runtime dynamic interpreter (`/lib/ld-linux-aarch64.so.1`) was missing from the root filesystem.
* **Resolution:** Recompile BusyBox with `CONFIG_STATIC=y` using `ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-`.

### 24.2 `networking/tc.c: error: 'TCA_CBQ_MAX' undeclared`
* **Root Cause:** The upstream Linux kernel permanently removed the deprecated Class Based Queueing (CBQ) scheduler in Linux 6.8+. Older BusyBox versions (e.g. 1.36.0) attempt to reference removed structs.
* **Resolution:** Disable the `tc` (Traffic Control) network utility in BusyBox:
  ```bash
  sed -i 's/CONFIG_TC=y/# CONFIG_TC is not set/' .config
  sed -i 's/CONFIG_FEATURE_TC_INGRESS=y/# CONFIG_FEATURE_TC_INGRESS is not set/' .config
  ```

### 24.3 `fatal error: generated/timeconst.h: No such file or directory`
* **Root Cause:** The Linux kernel build scripts defer generating constant timing multipliers until module/kernel build targets are invoked. Out-of-tree modules compiling against an unprepared kernel tree fail because this header is missing.
* **Resolution:** Run the kernel preparation targets before compiling external modules:
  ```bash
  cd linux
  ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make prepare
  ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make modules_prepare
  ```

### 24.4 `make[2]: *** No rule to make target 'scripts/module.lds', needed by 'test-module.ko'`
* **Root Cause:** ARM64 modules require an architecture-specific linker script (`scripts/module.lds`). Compiling only the core kernel (`make Image`) does not build module linker prerequisites.
* **Resolution:** Execute `make modules_prepare` in the kernel directory.

### 24.5 `Your display is too small to run Menuconfig!`
* **Root Cause:** `lxdialog` requires at least 80 columns by 19 lines of terminal display area.
* **Resolution:** Maximize the terminal pane or run non-interactive configuration targets such as `make defconfig`.

### 24.6 `cpio: not found`
* **Root Cause:** Minimal container base images (such as GitHub Codespaces default environments) do not include legacy packaging utilities by default.
* **Resolution:** Install using `sudo apt-get install -y cpio`.

---


## 25. Chat Follow-Up: Rebuilding the Kernel and Preparing for External Modules

This section records the additional troubleshooting and decisions made while following the workflow above in GitHub Codespaces. It is intended to preserve the exact lessons learned from the build attempts.

### 25.1 Important distinction: U-Boot `defconfig` vs Linux `defconfig`

One of the most important mistakes was using:

```bash
make qemu_arm64_defconfig
```

inside the **Linux kernel** directory.

`qemu_arm64_defconfig` is a valid configuration target for **U-Boot**, because U-Boot contains:

```text
u-boot/configs/qemu_arm64_defconfig
```

It is not necessarily a Linux kernel configuration target.

The Linux kernel has its own architecture-specific configuration targets. For a generic ARM64 Linux kernel, the simple choice is:

```bash
cd /workspaces/codespaces-blank/linux

export ARCH=arm64
export CROSS_COMPILE=/workspaces/codespaces-blank/arm-gnu-toolchain-13.3.rel1-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-

make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" defconfig
```

This creates:

```text
linux/.config
```

### 25.2 What `make mrproper` does

`make mrproper` performs a much more complete cleanup than `make clean`.

It can remove:

```text
.config
generated files
build artifacts
temporary configuration files
```

Therefore, after:

```bash
make mrproper
```

you normally need to configure the kernel again:

```bash
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" defconfig
```

before attempting a kernel build.

A useful rule is:

```text
make clean
    ↓
Remove compiled objects
Keep .config

make mrproper
    ↓
Deep cleanup
Remove .config too
    ↓
Run defconfig/menuconfig again
```

### 25.3 Why `make -j20` failed after `mrproper`

An attempted sequence was:

```bash
make mrproper
make qemu_arm64_defconfig
make -j20
```

inside the Linux source directory.

The configuration command failed because Linux could not find:

```text
arch/arm/configs/qemu_arm64_defconfig
```

Then:

```bash
make -j20
```

also failed because `.config` did not exist.

The important chain is:

```text
make mrproper
     ↓
.config deleted
     ↓
wrong configuration target
     ↓
qemu_arm64_defconfig fails
     ↓
no .config
     ↓
kernel build cannot start
```

The correct recovery is:

```bash
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" defconfig
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" -j20
```

### 25.4 Cross-compiler problems and how to recognize them

An earlier U-Boot build produced errors such as:

```text
cc1: error: bad value 'armv8-a+crc' for '-march=' switch
```

and showed valid `-march` values associated with x86-64.

This is a strong indication that the build was accidentally using the **host x86-64 compiler** instead of an ARM64 cross-compiler.

The host compiler:

```text
gcc
```

produces:

```text
x86-64 machine code
```

The ARM64 cross-compiler:

```text
aarch64-none-linux-gnu-gcc
```

produces:

```text
AArch64 / ARM64 machine code
```

The toolchain used in this workspace is located under:

```text
/workspaces/codespaces-blank/
└── arm-gnu-toolchain-13.3.rel1-x86_64-aarch64-none-linux-gnu/
    └── bin/
        ├── aarch64-none-linux-gnu-gcc
        ├── aarch64-none-linux-gnu-ld
        ├── aarch64-none-linux-gnu-ar
        └── ...
```

Therefore:

```bash
export CROSS_COMPILE=/workspaces/codespaces-blank/arm-gnu-toolchain-13.3.rel1-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-
```

Verify it before building:

```bash
${CROSS_COMPILE}gcc --version
${CROSS_COMPILE}gcc -dumpmachine
```

The second command should report something similar to:

```text
aarch64-none-linux-gnu
```

### 25.5 Why the `echogcc` error appeared

An incorrect `CROSS_COMPILE` environment variable previously caused an error resembling:

```text
aarch64-none-linux-gnu-echogcc: not found
```

This indicates that the cross-compiler prefix was being combined incorrectly by the build system.

Another check showed:

```text
bash: /home/codespace/.../aarch64-none-linux-gnu-gcc: No such file or directory
```

while the actual workspace was under:

```text
/workspaces/codespaces-blank/
```

The lesson is:

**Do not assume the toolchain path. Verify it.**

Useful commands:

```bash
pwd
```

```bash
ls /workspaces/codespaces-blank/arm-gnu-toolchain-13.3.rel1-x86_64-aarch64-none-linux-gnu/bin/
```

or:

```bash
find /workspaces -name aarch64-none-linux-gnu-gcc -type f 2>/dev/null
```

Then set `CROSS_COMPILE` to the directory that actually contains the compiler.

### 25.6 `aarch64-linux-gnu-` vs `aarch64-none-linux-gnu-`

There are two prefixes that appeared during the work:

```text
aarch64-linux-gnu-
aarch64-none-linux-gnu-
```

They are both commonly used AArch64 GNU toolchain naming conventions, but they are not interchangeable strings if only one corresponding compiler is installed.

For this particular workspace, the installed toolchain was identified as:

```text
aarch64-none-linux-gnu-gcc
```

Therefore the safest setting is the actual compiler prefix:

```bash
export CROSS_COMPILE=/workspaces/codespaces-blank/arm-gnu-toolchain-13.3.rel1-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-
```

Always verify with:

```bash
${CROSS_COMPILE}gcc --version
```

### 25.7 Correct kernel preparation sequence

For building an out-of-tree module, the kernel source tree must first have a valid configuration and the required generated files.

A reliable sequence is:

```bash
cd /workspaces/codespaces-blank/linux

export ARCH=arm64
export CROSS_COMPILE=/workspaces/codespaces-blank/arm-gnu-toolchain-13.3.rel1-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-

${CROSS_COMPILE}gcc --version
${CROSS_COMPILE}gcc -dumpmachine

make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" defconfig

make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" prepare

make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" modules_prepare
```

For a complete kernel build:

```bash
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" -j20
```

This should produce:

```text
arch/arm64/boot/Image
```

### 25.8 What happened when `make prepare` asked configuration questions

An attempt was made with:

```bash
cd /workspaces/codespaces-blank/linux

ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make prepare
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make modules_prepare
```

The kernel entered:

```text
Restart config...
```

and asked:

```text
ARMv8.3 architectural features

Enable support for pointer authentication (ARM64_PTR_AUTH) [Y/n/?]
```

Then:

```text
Use pointer authentication for kernel (ARM64_PTR_AUTH_KERNEL) [Y/n/?] (NEW)
```

At this point `Ctrl+C` was pressed, producing:

```text
make: *** ... Interrupt
```

### 25.9 Why `make prepare` asked questions

The important point is that `prepare` is not necessarily responsible for creating a complete kernel configuration from nothing.

If `.config` is missing, incomplete, or inconsistent with the source tree, the kernel's Kconfig system may invoke:

```text
syncconfig
```

and ask about newly introduced options.

The better approach is to establish a proper configuration first:

```bash
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" defconfig
```

Then run:

```bash
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" prepare
make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" modules_prepare
```

This avoids accidentally stopping in the middle of a configuration process.

### 25.10 What the `^C` actually means

The output:

```text
^C
```

means the process received an interrupt, normally because:

```text
Ctrl+C
```

was pressed.

Therefore:

```text
make: *** ... Interrupt
```

does **not** mean that the kernel source was corrupted.

It means the build/configuration process was manually interrupted.

After creating a valid `.config`, simply rerun the preparation commands.

### 25.11 The assembler warning

The following warning appeared:

```text
arch/arm64/Makefile:36: Detected assembler with broken .inst; disassembly will be unreliable
```

This warning is different from the interruption.

The build was stopped by:

```text
^C
```

The warning says that the assembler being detected has a limitation related to `.inst`, which affects the reliability of disassembly. It is not the direct reason the command stopped in this session.

The first priority is therefore:

1. Use the correct ARM64 cross-compiler.
2. Create a valid `.config`.
3. Run `prepare`.
4. Run `modules_prepare`.
5. Build the kernel/module.

If the assembler warning causes a later actual build failure, investigate the toolchain/binutils version separately.

### 25.12 Recommended complete recovery from the current state

If the Linux tree is currently in the state produced by the interrupted configuration, use:

```bash
cd /workspaces/codespaces-blank/linux

export ARCH=arm64
export CROSS_COMPILE=/workspaces/codespaces-blank/arm-gnu-toolchain-13.3.rel1-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-

${CROSS_COMPILE}gcc --version
${CROSS_COMPILE}gcc -dumpmachine

make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" defconfig

make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" prepare

make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" modules_prepare

make ARCH=arm64 CROSS_COMPILE="$CROSS_COMPILE" -j20
```

Then verify:

```bash
ls -lh arch/arm64/boot/Image
```

If `Image` exists, the kernel build stage is complete.

### 25.13 Building the external module after kernel preparation

Once the kernel has been configured/prepared:

```bash
cd /workspaces/codespaces-blank/kernel-module

make ARCH=arm64 \
     CROSS_COMPILE="$CROSS_COMPILE" \
     -C ../linux \
     M=$PWD \
     modules
```

Expected output:

```text
test-module.ko
```

Verify:

```bash
ls -lh test-module.ko
```

The key relationship is:

```text
                 Linux kernel source
                         │
              .config + generated files
                         │
                         ▼
                Kbuild / kernel build
                         │
                         ▼
                 test-module.ko
```

The module should be built against the kernel that will actually boot it.

### 25.14 Host PC vs QEMU guest vs Raspberry Pi

A recurring source of confusion is where files and commands exist.

#### 25.14.1 Host / Codespaces

This is where you perform the build:

```text
/workspaces/codespaces-blank/
├── linux/
├── u-boot/
├── busybox-1_36_0/
├── kernel-module/
├── initramfs/
└── initramfs.cpio.gz
```

Commands such as:

```bash
make
cp
find
cpio
```

are executed here while constructing the system.

#### 25.14.2 QEMU guest

When QEMU starts:

```bash
qemu-system-aarch64 ...
```

the ARM64 Linux kernel runs inside the virtual machine.

The `initramfs.cpio.gz` created on the host is passed to QEMU and unpacked into the guest's RAM.

Inside QEMU you see:

```text
/ # 
```

and commands such as:

```bash
ls
insmod
lsmod
rmmod
dmesg
```

operate inside the ARM64 guest.

#### 25.14.3 Raspberry Pi

The Raspberry Pi is a separate physical target.

Its SD card normally has:

```text
SD CARD
├── bootfs
│   ├── U-Boot
│   ├── Image
│   └── Device Tree
└── rootfs
    ├── bin/
    ├── etc/
    ├── lib/
    └── usr/
```

Rebuilding the Linux kernel does **not** automatically mean the SD card must be repartitioned or reformatted. If the partitions already exist, normally only the required boot files need to be replaced.

### 25.15 Why `initramfs/` is not created by Linux

The directory:

```text
initramfs/
```

is a directory constructed on the **host machine**.

It becomes the archive:

```text
initramfs.cpio.gz
```

using:

```bash
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
```

The flow is:

```text
Host
│
├── initramfs/
│   ├── init
│   ├── bin/
│   ├── dev/
│   ├── proc/
│   ├── sys/
│   └── usr/modules/test-module.ko
│
│        cpio + gzip
▼
initramfs.cpio.gz
│
│        QEMU -initrd
▼
ARM64 Linux RAM
│
▼
Temporary root filesystem
```

### 25.16 Kernel module development cycle

An efficient way to understand the module workflow is:

```text
Write / modify test-module.c
          │
          ▼
Cross-compile with Kbuild
          │
          ▼
     test-module.ko
          │
          ▼
Copy into initramfs
          │
          ▼
Rebuild initramfs.cpio.gz
          │
          ▼
Boot QEMU
          │
          ▼
insmod test-module.ko
          │
          ▼
Test behavior
          │
          ▼
rmmod test_module
          │
          └──────────────► modify source and repeat
```

This is why kernel modules are useful during driver development: the module can often be rebuilt and loaded without rebuilding the complete kernel.

### 25.17 QEMU shutdown in the minimal BusyBox environment

The minimal `/init` used in this project is:

```sh
mount -t devtmpfs devtmpfs /dev
mount -t proc none /proc
mount -t sysfs none /sys
exec /bin/sh
```

There is no full system manager such as `systemd`, and therefore:

```bash
poweroff
```

may not work as it would on a normal Linux distribution.

The reliable QEMU escape sequence when using:

```text
-nographic
```

is:

```text
Ctrl+A
release
X
```

This exits QEMU.

Another method is:

```text
Ctrl+A
release
C
```

to enter the QEMU monitor, followed by:

```text
quit
```

### 25.18 Final checklist

Before building the module, verify:

```bash
# 1. Correct directory
pwd
# /workspaces/codespaces-blank/linux

# 2. Correct compiler
${CROSS_COMPILE}gcc --version
${CROSS_COMPILE}gcc -dumpmachine

# 3. Kernel configuration exists
ls -l .config

# 4. Kernel preparation completed
ls -l include/generated/

# 5. Module preparation completed
ls -l scripts/module.lds

# 6. Kernel image exists if the full kernel was built
ls -lh arch/arm64/boot/Image
```

Then:

```bash
cd /workspaces/codespaces-blank/kernel-module

make ARCH=arm64 \
     CROSS_COMPILE="$CROSS_COMPILE" \
     -C ../linux \
     M=$PWD \
     modules
```

Expected final module:

```text
/workspaces/codespaces-blank/kernel-module/test-module.ko
```

---

## 26. Compact Mental Model

The entire project can be remembered as five layers:

```text
┌───────────────────────────────────────────────┐
│                 USER SPACE                    │
│ BusyBox → sh, ls, insmod, lsmod, rmmod       │
├───────────────────────────────────────────────┤
│               INITRAMFS                      │
│ /init + /dev + /proc + /sys + test-module.ko │
├───────────────────────────────────────────────┤
│             LINUX KERNEL                     │
│ ARM64 Image + kernel configuration            │
├───────────────────────────────────────────────┤
│                U-BOOT                        │
│ Loads kernel / DTB and starts Linux           │
├───────────────────────────────────────────────┤
│              QEMU / HARDWARE                 │
│ ARM64 CPU + RAM + UART + virtual hardware     │
└───────────────────────────────────────────────┘
```

And the build process is:

```text
Cross-compiler
      │
      ├──────────────► U-Boot ──────────► u-boot.bin
      │
      ├──────────────► Linux ───────────► Image
      │                    │
      │                    └────────────► prepared kernel tree
      │                                      │
      │                                      ▼
      │                               test-module.ko
      │
      └──────────────► BusyBox ─────────► _install/
                                             │
                                             ▼
                                      initramfs/
                                             │
                                   cpio + gzip
                                             │
                                             ▼
                                      initramfs.cpio.gz
                                             │
                                             ▼
                                           QEMU
                                             │
                                             ▼
                                     ARM64 Linux shell
                                             │
                                      insmod / rmmod
```

The most important troubleshooting principle from the session is:

> **Always distinguish the host architecture, target architecture, build directory, configuration file, and runtime environment.**

Most of the errors encountered were caused by one of these boundaries being mixed up.

---

## 27. Device Drivers, `/dev`, and `/sys`

A **device driver** is software meant to interact with a specific piece of hardware. Device drivers hide the underlying hardware-specific interactions from user space.

### 27.1 What is a Device Driver?

A device driver is software inside the Linux kernel that knows how to communicate with a particular type of hardware.

```text
User application
      │
      ▼
Linux interface
/dev or /sys
      │
      ▼
Device driver
      │
      ▼
Hardware
```

The application does not need to know which hardware registers to access, how to handle device-specific commands, interrupts, or data transfers. The driver handles those details.

```text
Application
     │
     │ "Read this"
     ▼
Linux
     │
     ▼
Device driver
     │
     │ hardware-specific operations
     ▼
Hardware
```

This is what it means to say that device drivers **hide the underlying interactions with the hardware from the user**.

### 27.2 Where are Linux Device Drivers?

The Linux kernel source contains a major directory called:

```text
drivers/
```

It contains source code for many categories of device drivers, including:

```text
drivers/
├── block/       → block devices
├── char/        → character devices
├── gpio/        → GPIO
├── input/       → input devices
├── media/       → media devices
├── mtd/         → flash memory
├── net/         → networking
├── pci/         → PCI devices
├── serial/      → serial/UART devices
├── usb/         → USB devices
├── video/       → video/display
└── ...
```

Device drivers make up a very large portion of the Linux kernel source tree. The course material notes that the `drivers` directory is the largest directory and is approximately 69% of the Linux kernel source code.

### 27.3 `/dev` — The Device Interface

User space can interact with many device drivers through:

```text
/dev
```

`/dev` is a special filesystem containing **device files**.

For example:

```bash
ls /dev
```

may show entries such as:

```text
console
ttyAMA0
sda
sda1
sda2
```

These are not ordinary files such as `hello.txt` or `program.c`. A device file provides an interface to a device or kernel subsystem.

### 27.4 Why Do Devices Look Like Files?

Linux provides a consistent file-oriented interface for many resources. User space can use operations such as:

```text
open()
read()
write()
close()
```

A regular file and a device file can therefore be accessed through similar user-space operations even though their underlying implementations are very different.

For example:

```bash
echo "Hello from userspace" > /dev/console
```

Conceptually:

```text
echo
 │
 │ write "Hello"
 ▼
/dev/console
 │
 ▼
Console device interface
 │
 ▼
Kernel console subsystem/driver
 │
 ▼
QEMU terminal
```

This demonstrates how a user-space program can communicate with kernel/device functionality through a device file.

### 27.5 Flash Drive Example

If Linux detects a flash drive with two partitions, they could appear as:

```text
/dev/sdX
├── /dev/sdX1
└── /dev/sdX2
```

Here:

```text
sd  → disk/block-device naming
X   → device letter chosen by Linux
1   → partition 1
2   → partition 2
```

For example, the actual device could be:

```text
/dev/sda
├── /dev/sda1
└── /dev/sda2
```

The exact letter depends on which devices Linux detects.

A partition can then be mounted, for example:

```bash
mount /dev/sda1 /mnt
```

Conceptually:

```text
Flash drive
      │
      ▼
/dev/sda1
      │
      ▼
Storage/block driver
      │
      ▼
Block layer
      │
      ▼
Filesystem
      │
      ▼
/mnt
```

### 27.6 `/dev` Is Not an Ordinary Directory

Although `/dev` looks like a normal directory when you run `ls /dev`, its entries are device files provided through a special filesystem. They do not simply contain the raw device data as ordinary files.

```text
/dev entry
    │
    ▼
Kernel interface
    │
    ▼
Device driver
    │
    ▼
Hardware
```

### 27.7 `/sys` — sysfs

Another important interface is:

```text
/sys
```

`/sys` is also known as **sysfs**. It is a **virtual filesystem** used by the Linux kernel to expose information and attributes about hardware and kernel objects to user space.

A useful mental model is:

```text
/dev
  ↓
"Interact with the device"

/sys
  ↓
"Inspect information about the device"
```

This is a simplified mental model; the exact interfaces provided by Linux are more detailed.

Conceptually:

```text
User space
    │
    │ ls /sys
    │ cat /sys/...
    ▼
  sysfs
    │
    ▼
Linux kernel objects
    │
    ▼
Drivers
    │
    ▼
Hardware
```

### 27.8 Why Is `/sys` Different from a Normal Directory?

The contents of sysfs are controlled by the kernel and its subsystems/drivers. Users do not simply create arbitrary files there like they would in a normal filesystem.

Instead, device drivers follow defined Linux kernel APIs to expose attributes through sysfs.

```text
Driver
  │
  │ exposes attribute
  ▼
sysfs
  │
  │ user reads attribute
  ▼
cat
```

### 27.9 `/sys/class`

An especially useful part of sysfs is:

```text
/sys/class
```

It organizes devices according to their device class. For example:

```text
/sys/class/
├── block/
├── gpio/
├── net/
├── tty/
├── mtd/
└── ...
```

The course focuses on:

```text
/sys/class/mtd
```

### 27.10 What Is MTD?

MTD means **Memory Technology Device**. It is a Linux subsystem for certain types of flash memory, such as raw NAND and NOR flash.

In the QEMU environment used in the course, a virtualized flash device is exposed through the MTD subsystem.

### 27.11 Exploring `/sys/class/mtd`

Navigate to:

```bash
cd /sys/class/mtd
ls
```

You may see:

```text
mtd0
```

Conceptually:

```text
/sys/class/mtd
       │
       ▼
     mtd0
       │
       ▼
Virtual flash device
```

`mtd0` represents the first MTD device exposed by the kernel.

Explore it further:

```bash
ls /sys/class/mtd/mtd0
```

The exact attributes exposed can depend on the kernel configuration and device.

### 27.12 Reading the Flash Device Size

One useful attribute is `size`:

```bash
cat /sys/class/mtd/mtd0/size
```

Conceptually:

```text
cat
 │
 ▼
/sys/class/mtd/mtd0/size
 │
 ▼
MTD subsystem
 │
 ▼
Flash device information
 │
 ▼
size
```

The returned value represents the device size in bytes. For example, `16777216` bytes is 16 MiB.

### 27.13 `/dev` vs `/sys`

This distinction is important:

| Interface | Main purpose | Example |
|---|---|---|
| `/dev` | Device access/interface | `/dev/console` |
| `/sys` | Device/kernel information and attributes | `/sys/class/mtd/mtd0/size` |

A useful mental model is:

```text
                 Linux Kernel
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
        /dev                    /sys
          │                       │
          │                       │
   "Use/interact             "Inspect device
     with device"              information"
          │                       │
          ▼                       ▼
       Driver                  Driver
          │                       │
          └───────────┬───────────┘
                      ▼
                   Hardware
```

### 27.14 Complete Device-Driver Picture

Putting the pieces together:

```text
                     USER SPACE
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
          /dev/console        /sys/class/mtd
              │                     │
              │                     ▼
              │                MTD attributes
              │                     │
              ▼                     │
       Device interface             │
              │                     │
              └──────────┬──────────┘
                         ▼
                    Linux Driver
                         │
                         ▼
                  Kernel subsystem
                         │
                         ▼
                      Hardware
```

The device driver is the critical middle layer between Linux and the hardware.

### 27.15 Why Applications Do Not Directly Access Hardware

Without drivers, an application would need to know details such as:

```text
Which register?
Which address?
Which command?
Which timing?
Which interrupt?
Which controller?
Which hardware revision?
```

Instead:

```text
Application
     │
     │ standard Linux interface
     ▼
Kernel
     │
     │ device driver
     ▼
Hardware
```

The driver absorbs the hardware-specific complexity, allowing user-space applications to use standard Linux interfaces without understanding the hardware implementation.

### 27.16 Connection to Device Tree

The Device Tree describes hardware to the Linux kernel:

```text
Device Tree
     │
     │ describes hardware
     ▼
Linux Kernel
     │
     ▼
Driver
     │
     ▼
Hardware
     │
     ▼
/dev and /sys
     │
     ▼
User Space
```

For example:

```text
Device Tree
    │
    │ "There is an MTD/flash device"
    ▼
Linux
    │
    ▼
MTD driver
    │
    ├──────────────► /dev/...
    │
    └──────────────► /sys/class/mtd/...
```

So Device Tree, device drivers, `/dev`, and `/sys` are connected parts of the same embedded Linux hardware model.

### 27.17 Key Points to Remember

**Device driver**

> Software in the Linux kernel that knows how to communicate with a particular device or hardware subsystem.

**`drivers/`**

> A major Linux kernel source directory containing source code for many device drivers.

**`/dev`**

> A special filesystem exposing device files that provide interfaces for interacting with devices.

**`/sys` / sysfs**

> A virtual filesystem through which the kernel and drivers expose information and attributes about hardware and kernel objects to user space.

**MTD**

> Linux's Memory Technology Device subsystem for certain types of flash memory.

### 27.18 One Mental Model to Memorize

```text
              User Space
                  │
          ┌───────┴───────┐
          ▼               ▼
        /dev             /sys
          │               │
          │               │
          ▼               ▼
        Device          Hardware
       interface       information
          │               │
          └───────┬───────┘
                  ▼
             Device Driver
                  │
                  ▼
               Hardware
```

> **The device driver is the bridge between Linux and the hardware.**
