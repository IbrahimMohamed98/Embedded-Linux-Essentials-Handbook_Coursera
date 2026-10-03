# U-Boot Complete Study Guide

# Table of Contents

1.  [What Is U-Boot?](#1-what-is-u-boot)
2.  [U-Boot Source Tree](#2-u-boot-source-tree)
3.  [U-Boot Configuration](#3-u-boot-configuration)
    -   [Kconfig](#31-what-is-defconfig)
    -   [Kconfig vs Device Tree](#32-kconfig-vs-device-tree)
4.  [Cross Compilation](#4-cross-compilation)
5.  [Building U-Boot](#5-building-u-boot)
    -   [Important Build Outputs](#51-important-build-outputs)
6.  [U-Boot Driver Model](#6-u-boot-driver-model)
    -   [What Is a UCLASS?](#61-what-is-a-uclass)
    -   [Driver](#62-driver)
    -   [Device](#63-device)
    -   [Device Tree](#64-device-tree)
7.  [Registering a U-Boot Driver](#7-registering-a-u-boot-driver)
    -   [`of_match`](#71-of_match)
    -   [`ops`](#72-ops)
    -   [`probe()`](#73-probe)
    -   [`bind()`](#74-bind)
8.  [U-Boot Environment](#8-u-boot-environment)
9.  [Important Environment
    Variables](#9-important-environment-variables)
10. [`printenv`](#10-printenv)
11. [`setenv`](#11-setenv)
12. [`saveenv`](#12-saveenv)
13. [RAM Environment vs Persistent
    Environment](#13-ram-environment-vs-persistent-environment)
14. [The `boot` Command](#14-the-boot-command)
15. [U-Boot Commands](#15-u-boot-commands)
16. [The `cmd/` Directory](#16-the-cmd-directory)
17. [`U_BOOT_CMD()`](#17-u_boot_cmd)
18. [`struct cmd_tbl`](#18-struct-cmd_tbl)
19. [How a U-Boot Command Is
    Executed](#19-how-a-u-boot-command-is-executed)
20. [Creating a Custom U-Boot
    Command](#20-creating-a-custom-u-boot-command)
21. [Create `cmd/packt.c`](#21-create-cmdpacktc)
22. [Understanding `do_packt()`](#22-understanding-do_packt)
23. [Understanding the Function
    Arguments](#23-understanding-the-function-arguments)
24. [`return 0`](#24-return-0)
25. [Understanding `U_BOOT_CMD()`
    Parameters](#25-understanding-u_boot_cmd-parameters)
26. [Modify `cmd/Makefile`](#26-modify-cmdmakefile)
27. [Rebuild U-Boot](#27-rebuild-u-boot)
28. [Test the Custom Command](#28-test-the-custom-command)
29. [U-Boot on Raspberry Pi 5](#29-u-boot-on-raspberry-pi-5)
30. [Raspberry Pi Boot Media](#30-raspberry-pi-boot-media)
31. [UART and the Raspberry Pi Debug
    Probe](#31-uart-and-the-raspberry-pi-debug-probe)
32. [U-Boot Console](#32-u-boot-console)
33. [QEMU: Running U-Boot Without Physical
    Hardware](#33-qemu-running-u-boot-without-physical-hardware)
34. [Configure U-Boot for QEMU
    ARM64](#34-configure-u-boot-for-qemu-arm64)
35. [Launch U-Boot in QEMU](#35-launch-u-boot-in-qemu)
36. [QEMU Command-Line Arguments](#36-qemu-command-line-arguments)
37. [`-M virt`](#371--m-virt)
38. [`-cpu cortex-a76`](#372--cpu-cortex-a76)
39. [`-bios u-boot.bin`](#373--bios-u-bootbin)
40. [`-m 2G`](#374--m-2g)
41. [`-nographic`](#375--nographic)
42. [QEMU Startup Flow](#42-qemu-startup-flow)
43. [Exiting QEMU](#43-exiting-qemu)
44. [QEMU Environment Persistence](#44-qemu-environment-persistence)
45. [Common Mistakes](#45-common-mistakes)
46. [Important Concepts to Memorize](#46-important-concepts-to-memorize)
47. [Command Cheat Sheet](#47-command-cheat-sheet)
48. [Build Cheat Sheet](#48-build-cheat-sheet)
49. [Custom Command Cheat Sheet](#49-custom-command-cheat-sheet)
50. [One-Minute Revision](#50-one-minute-revision)
51. [Self-Test Questions](#51-self-test-questions)
52. [Final U-Boot Mental Model](#52-final-u-boot-mental-model)

------------------------------------------------------------------------

## 1. What Is U-Boot?

U-Boot is a **bootloader** commonly used in embedded systems.

Its job is to initialize enough of the hardware to continue the boot
process, load the operating system and related files, and provide a
command-line interface for development and debugging.

A simplified boot flow is:

``` text
Power On
   │
   ▼
Boot ROM / Firmware
   │
   ▼
U-Boot
   │
   ├── Initialize hardware
   ├── Read configuration
   ├── Find/load kernel
   ├── Load Device Tree
   └── Start Linux
          │
          ▼
       Linux Kernel
```

U-Boot is therefore an important layer between the board's initial
startup code and the operating system.

------------------------------------------------------------------------

# 2. U-Boot Source Tree

The U-Boot source tree contains different parts of the bootloader.

Some important directories are:

  Directory                    Purpose
  ---------------------------- --------------------------------
  `cmd/`                       U-Boot command implementations
  `drivers/`                   Device drivers
  `configs/`                   Board/target defconfig files
  `include/`                   Header files
  `common/`                    Common U-Boot functionality
  `arch/`                      Architecture-specific code
  `board/`                     Board-specific code
  `dts/` / Device Tree files   Hardware description

A useful mental model is:

``` text
U-Boot Source Tree
       │
       ├── cmd/       → Commands
       ├── drivers/   → Hardware drivers
       ├── configs/   → Default configurations
       ├── arch/      → CPU architecture code
       ├── board/     → Board-specific code
       └── include/   → Headers
```

The `drivers/` directory is especially important because U-Boot contains
a large number of hardware drivers.

------------------------------------------------------------------------

# 3. U-Boot Configuration

U-Boot uses the **Kconfig** configuration system.

Kconfig allows features to be enabled or disabled when configuring the
build.

Examples of things that can be configured include:

-   Networking
-   Logging
-   Commands
-   Drivers
-   Storage support
-   Other optional software features

The general relationship is:

``` text
Kconfig
   │
   ▼
defconfig
   │
   ▼
.config
   │
   ▼
Build
```

------------------------------------------------------------------------

## 3.1 What Is `defconfig`?

A `defconfig` is a predefined configuration for a particular target.

Examples:

``` text
configs/rpi_arm64_defconfig
configs/qemu_arm64_defconfig
```

For example:

``` bash
make rpi_arm64_defconfig
```

selects the default configuration for the Raspberry Pi ARM64 target.

This generates the full:

``` text
.config
```

file.

The important distinction is:

``` text
defconfig
    │
    └── Small set of target-specific defaults

.config
    │
    └── Full configuration used by the build
```

------------------------------------------------------------------------

## 3.2 Kconfig vs Device Tree

These two are easy to confuse.

### Kconfig

Kconfig describes **software/build configuration**.

For example:

``` text
Should networking support be compiled?
Should a particular driver be included?
Should logging be enabled?
```

### Device Tree

Device Tree describes **hardware information/configuration**.

For example:

``` text
Which GPIO is connected to this button?
Which I2C controller exists?
What address does this device use?
```

A useful rule is:

``` text
Kconfig
   ↓
Software features / build configuration

Device Tree
   ↓
Hardware description
```

A common mistake is trying to use Kconfig to describe fixed
board-specific hardware such as GPIO pins.

------------------------------------------------------------------------

# 4. Cross Compilation

U-Boot is commonly built on one machine and executed on another
architecture.

For example:

``` text
Development PC
x86-64
   │
   │ cross compiler
   ▼
U-Boot for ARM64
   │
   ▼
Raspberry Pi 5
```

This is called **cross-compilation**.

The compiler runs on the development machine but produces code for the
target architecture.

For ARM64, an example toolchain prefix is:

``` text
aarch64-none-linux-gnu-
```

The corresponding compiler may be:

``` text
aarch64-none-linux-gnu-gcc
```

The U-Boot build system can be told to use the toolchain with:

``` bash
export CROSS_COMPILE=aarch64-none-linux-gnu-
```

The important idea is:

``` text
Host architecture ≠ Target architecture
```

------------------------------------------------------------------------

# 5. Building U-Boot

The general U-Boot build process is:

``` text
Install dependencies
        │
        ▼
Select target defconfig
        │
        ▼
Generate .config
        │
        ▼
Run make
        │
        ▼
U-Boot binaries
```

For Raspberry Pi 5:

``` bash
make rpi_arm64_defconfig
make
```

For QEMU ARM64:

``` bash
make qemu_arm64_defconfig
make
```

The exact dependencies are documented by U-Boot, including
`doc/build/gcc.rst`.

------------------------------------------------------------------------

## 5.1 Important Build Outputs

A U-Boot build can produce several important files.

  File            Purpose
  --------------- ----------------------------------------
  `u-boot`        U-Boot executable/debug build
  `u-boot.bin`    Binary U-Boot image
  `u-boot.sym`    Symbol information
  `u-boot.map`    Linker/memory layout information
  `System.map`    Symbol/map information
  `u-boot.cfg`    Final build configuration information
  `u-boot.dtb`    U-Boot Device Tree Blob when generated
  `u-boot.srec`   S-Record output when configured

For many practical board/QEMU workflows, the important file is:

``` text
u-boot.bin
```

------------------------------------------------------------------------

# 6. U-Boot Driver Model

U-Boot has a **Driver Model (DM)** that provides a common framework for
managing hardware devices and their drivers.

The important concepts are:

``` text
Driver Model
     │
     ├── UCLASS
     │
     ├── Driver
     │
     └── Device
```

------------------------------------------------------------------------

## 6.1 What Is a UCLASS?

A **UCLASS** groups similar drivers under a common interface/category.

For example:

``` text
UCLASS_I2C
    │
    ├── I2C Driver A
    ├── I2C Driver B
    └── I2C Driver C
```

Another example:

``` text
UCLASS_BUTTON
    │
    ├── GPIO Button Driver
    └── Other Button Driver
```

A useful analogy is:

``` text
UCLASS ≈ Interface / abstract category
Driver  ≈ Implementation
Device  ≈ Actual instance
```

------------------------------------------------------------------------

## 6.2 Driver

A driver contains the hardware-specific implementation needed to
communicate with a device.

For example, a GPIO button driver knows how to obtain the state of a
button connected through GPIO hardware.

The driver is registered with U-Boot's Driver Model.

------------------------------------------------------------------------

## 6.3 Device

A device represents an actual device instance.

For example:

``` text
UCLASS_BUTTON
      │
      ▼
GPIO button device
      │
      ▼
Specific Device Tree node
```

------------------------------------------------------------------------

## 6.4 Device Tree

The Device Tree describes hardware to software.

For example, a Device Tree node may contain:

``` text
compatible = "gpio-keys"
```

U-Boot can use the `compatible` string to determine which driver should
handle the device.

Conceptually:

``` text
Device Tree
     │
     │ compatible
     ▼
Driver matching
     │
     ▼
Correct driver
     │
     ▼
Device instance
```

------------------------------------------------------------------------

# 7. Registering a U-Boot Driver

A driver can be registered using the `U_BOOT_DRIVER()` macro.

A simplified example is:

``` c
U_BOOT_DRIVER(button_gpio) = {
    .id = UCLASS_BUTTON,
    ...
};
```

The important part is:

``` c
.id = UCLASS_BUTTON
```

This associates the driver with the button UCLASS.

A more complete driver can contain fields such as:

``` text
U_BOOT_DRIVER(...)
     │
     ├── .id
     ├── .of_match
     ├── .ops
     ├── .priv_auto
     ├── .bind
     ├── .probe
     └── .remove
```

------------------------------------------------------------------------

## 7.1 `of_match`

`of_match` is used to match Device Tree compatible strings.

Conceptually:

``` text
Device Tree
    │
    │ "gpio-keys"
    ▼
of_match
    │
    ▼
GPIO button driver
```

------------------------------------------------------------------------

## 7.2 `ops`

`ops` provides the operations that the driver supports.

For example, a button driver may provide operations related to reading
button state.

``` text
UCLASS_BUTTON
      │
      ▼
     ops
      │
      ├── get_state()
      └── get_code()
```

------------------------------------------------------------------------

## 7.3 `probe()`

`probe()` is used to initialize a device when it is actually
needed/activated.

Conceptually:

``` text
Device found
    │
    ▼
Driver selected
    │
    ▼
probe()
    │
    ▼
Device initialized
```

------------------------------------------------------------------------

## 7.4 `bind()`

`bind()` is associated with binding a driver/device into the Driver
Model and performing preliminary setup.

A simplified mental model is:

``` text
bind()
  ↓
Associate device/driver with DM

probe()
  ↓
Initialize device for use
```

------------------------------------------------------------------------

# 8. U-Boot Environment

U-Boot uses **environment variables** to store configurable values and
state.

The basic form is:

``` text
NAME=VALUE
```

For example:

``` text
bootdelay=2
bootcmd=...
bootfile=...
```

Environment variables are useful because boot behavior can be changed
without recompiling U-Boot.

------------------------------------------------------------------------

# 9. Important Environment Variables

  Variable      Purpose
  ------------- -----------------------------------------------
  `bootcmd`     Commands used for the normal boot process
  `bootargs`    Kernel command-line arguments passed to Linux
  `bootdelay`   Delay before automatic boot
  `bootfile`    Kernel image filename/path
  `fdtfile`     Device Tree filename/path

A useful mental model is:

``` text
U-Boot Environment
        │
        ├── bootcmd
        ├── bootargs
        ├── bootdelay
        ├── bootfile
        └── fdtfile
```

------------------------------------------------------------------------

# 10. Environment Initialization and Storage

U-Boot needs to initialize and load its environment.

Conceptually:

``` text
U-Boot starts
      │
      ▼
Environment initialization
      │
      ▼
Try to load environment
      │
      ├── Valid environment
      │       │
      │       ▼
      │    Use it
      │
      └── Invalid/missing
              │
              ▼
       Use compiled defaults
```

Important concepts include:

-   `env_init()` initializes environment storage locations.
-   `env_load()` attempts to load the environment.
-   Environment backends can be registered for different storage types.
-   Imported environment data can be validated, including CRC checking.
-   If the stored environment is invalid, U-Boot can fall back to
    compiled-in defaults.

The actual storage depends on the target configuration.

Possible storage mechanisms include:

``` text
Flash
eMMC
FAT
Other configured persistent storage
```

------------------------------------------------------------------------

# 11. `printenv`

The command:

``` bash
printenv
```

displays the current environment variables.

Example:

``` text
U-Boot> printenv
bootdelay=2
bootcmd=...
bootfile=...
```

The exact variables and values depend on the target.

------------------------------------------------------------------------

# 12. `setenv`

The command:

``` bash
setenv NAME VALUE
```

changes an environment variable.

Example:

``` bash
setenv bootdelay 5
```

Before:

``` text
bootdelay=2
```

After:

``` text
bootdelay=5
```

The important distinction is:

``` text
setenv
  │
  ▼
Changes current environment
```

It does not automatically mean the change is permanently stored.

------------------------------------------------------------------------

# 13. `saveenv`

The command:

``` bash
saveenv
```

attempts to save the current environment to the configured persistent
environment storage.

Conceptually:

``` text
setenv
   │
   ▼
Environment in RAM
   │
   │ saveenv
   ▼
Persistent storage
```

Persistence depends on the U-Boot environment-storage configuration.

------------------------------------------------------------------------

# 14. RAM Environment vs Persistent Environment

### Temporary environment

``` text
setenv
   │
   ▼
RAM
   │
   │ reboot
   ▼
Changes may be lost
```

### Persistent environment

``` text
setenv
   │
   ▼
RAM
   │
   │ saveenv
   ▼
Persistent storage
   │
   │ reboot
   ▼
Environment restored
```

This distinction is important when testing U-Boot.

------------------------------------------------------------------------

# 15. The `boot` Command

The:

``` bash
boot
```

command starts the U-Boot boot process.

U-Boot commonly uses the `bootcmd` environment variable to describe what
should happen during normal boot.

Conceptually:

``` text
U-Boot> boot
       │
       ▼
    bootcmd
       │
       ▼
Boot commands
       │
       ▼
Load kernel / Device Tree / other files
       │
       ▼
Start operating system
```

------------------------------------------------------------------------

# 16. U-Boot Commands

Commands are compiled functionality provided by U-Boot.

Examples:

``` text
help
printenv
setenv
saveenv
boot
```

They are implemented in the U-Boot source tree.

A useful distinction is:

``` text
U-Boot command
      │
      └── Compiled functionality

Environment variable
      │
      └── Configurable data/state
```

------------------------------------------------------------------------

# 17. The `cmd/` Directory

U-Boot command implementations are commonly located under:

``` text
cmd/
```

For example:

``` text
cmd/
 ├── bootm.c
 ├── ...
 └── packt.c
```

A command generally has:

``` text
Command implementation
        +
Command registration
        +
Build-system entry
```

------------------------------------------------------------------------

# 18. `U_BOOT_CMD()`

U-Boot provides the:

``` c
U_BOOT_CMD()
```

macro to register a command.

Conceptually:

``` text
User types command
       │
       ▼
U-Boot command table
       │
       ▼
Registered command
       │
       ▼
Implementation function
```

For example:

``` c
U_BOOT_CMD(
    packt, 1, 1, do_packt,
    "Simple Packt command",
    ""
);
```

This connects:

``` text
"packt"
   │
   ▼
do_packt()
```

------------------------------------------------------------------------

# 19. `struct cmd_tbl`

U-Boot uses command-table information to describe commands.

A command entry contains information such as:

  ---------------------------------------------------------------------
  Field                              Purpose
  ---------------------------------- ----------------------------------
  `name`                             Command name

  `maxargs`                          Maximum number of arguments

  `repeatable` / command repeat      Whether the command can be
  information                        repeated

  `cmd`                              Function implementing the command

  `usage`                            Short help text

  `help`                             Detailed help

  `complete`                         Autocomplete callback
  ---------------------------------------------------------------------

Conceptually:

``` text
Command Table Entry
        │
        ├── Name
        ├── Argument information
        ├── Function
        ├── Usage
        ├── Help
        └── Completion
```

------------------------------------------------------------------------

# 20. How a U-Boot Command Is Executed

Suppose the user enters:

``` text
U-Boot> packt
```

The conceptual process is:

``` text
User input
    │
    ▼
"packt"
    │
    ▼
Find registered command
    │
    ▼
Command table entry
    │
    ▼
do_packt()
    │
    ▼
Command output
```

This is the key relationship to remember:

``` text
Command name
     │
     ▼
U_BOOT_CMD()
     │
     ▼
Function
```

------------------------------------------------------------------------

# 21. Creating a Custom U-Boot Command

We can modify U-Boot itself by adding a custom command.

The goal is:

``` text
U-Boot> packt
```

to print:

``` text
Hello Packt World
```

The complete flow is:

``` text
cmd/packt.c
     │
     ▼
do_packt()
     │
     ▼
U_BOOT_CMD()
     │
     ▼
cmd/Makefile
     │
     ▼
make
     │
     ▼
u-boot.bin
     │
     ▼
U-Boot
     │
     ▼
packt
     │
     ▼
Hello Packt World
```

------------------------------------------------------------------------

# 22. Create `cmd/packt.c`

Create:

``` text
cmd/packt.c
```

Example:

``` c
#include <command.h>
#include <stdio.h>

static int do_packt(struct cmd_tbl *cmdtp, int flag,
                    int argc, char *const argv[])
{
    printf("Hello Packt World\n");
    return 0;
}

U_BOOT_CMD(
    packt, 1, 1, do_packt,
    "Simple Packt command",
    ""
);
```

------------------------------------------------------------------------

# 23. Understanding `do_packt()`

The function is:

``` c
static int do_packt(struct cmd_tbl *cmdtp, int flag,
                    int argc, char *const argv[])
{
    printf("Hello Packt World\n");
    return 0;
}
```

It contains the actual behavior of the command.

When:

``` text
U-Boot> packt
```

is executed, U-Boot eventually calls:

``` c
do_packt()
```

which prints:

``` text
Hello Packt World
```

------------------------------------------------------------------------

# 24. Understanding the Function Arguments

### `struct cmd_tbl *cmdtp`

Contains information about the registered command.

### `int flag`

Contains command execution flags.

### `int argc`

Contains the number of command-line arguments.

For:

``` text
U-Boot> packt
```

conceptually:

``` text
argc = 1
argv[0] = "packt"
```

For:

``` text
U-Boot> packt hello
```

conceptually:

``` text
argc = 2

argv[0] = "packt"
argv[1] = "hello"
```

### `char *const argv[]`

Contains the command-line arguments.

------------------------------------------------------------------------

# 25. `return 0`

The function ends with:

``` c
return 0;
```

This indicates successful execution.

``` text
do_packt()
   │
   ├── Print message
   │
   └── return 0
           │
           ▼
        Success
```

------------------------------------------------------------------------

# 26. Understanding `U_BOOT_CMD()` Parameters

The example is:

``` c
U_BOOT_CMD(
    packt, 1, 1, do_packt,
    "Simple Packt command",
    ""
);
```

The important parameters are:

``` text
U_BOOT_CMD(
    name,
    maxargs,
    repeatable,
    command_function,
    usage,
    help
);
```

For our command:

  Parameter    Value                      Meaning
  ------------ -------------------------- ------------------------------------------------
  Name         `packt`                    Command typed by the user
  `maxargs`    `1`                        Maximum argument count for this simple command
  Repeatable   `1`                        Command is repeatable
  Function     `do_packt`                 Function that executes the command
  Usage        `"Simple Packt command"`   Short help
  Help         `""`                       Detailed help text

One important detail:

``` text
maxargs = 1
```

means the command itself is counted in the argument count.

------------------------------------------------------------------------

# 27. Modify `cmd/Makefile`

Creating:

``` text
cmd/packt.c
```

is not enough.

The build system also needs to compile the new source file.

The command's object file needs to be included in:

``` text
cmd/Makefile
```

Conceptually:

``` text
cmd/packt.c
     │
     ▼
  packt.o
     │
     ▼
cmd/Makefile
     │
     ▼
U-Boot build
```

If the Makefile is not updated appropriately, the source file may not be
compiled into the final U-Boot binary.

------------------------------------------------------------------------

# 28. Rebuild U-Boot

After changing the source code, rebuild:

``` bash
make
```

The important workflow is:

``` text
Modify source
     │
     ▼
Modify build system if necessary
     │
     ▼
make
     │
     ▼
New U-Boot binary
```

If you forget to rebuild, you may still be running the old `u-boot.bin`.

------------------------------------------------------------------------

# 29. Test the Custom Command

After booting the newly built U-Boot:

``` text
U-Boot> help
```

Check that `packt` appears.

Then:

``` text
U-Boot> help packt
```

Finally:

``` text
U-Boot> packt
```

Expected output:

``` text
Hello Packt World
```

The complete process is:

``` text
Source code
    │
    ▼
cmd/packt.c
    │
    ▼
do_packt()
    │
    ▼
U_BOOT_CMD()
    │
    ▼
cmd/Makefile
    │
    ▼
make
    │
    ▼
u-boot.bin
    │
    ▼
Run U-Boot
    │
    ▼
packt
    │
    ▼
Hello Packt World
```

------------------------------------------------------------------------

# 30. U-Boot on Raspberry Pi 5

U-Boot can run on physical hardware such as the Raspberry Pi 5.

For the Raspberry Pi ARM64 target:

``` bash
make rpi_arm64_defconfig
make
```

This uses:

``` text
configs/rpi_arm64_defconfig
```

and generates:

``` text
.config
```

followed by the U-Boot build.

The important output is:

``` text
u-boot.bin
```

------------------------------------------------------------------------

# 31. Raspberry Pi Boot Media

A typical Raspberry Pi OS SD-card setup contains boot and root
filesystem areas.

Conceptually:

``` text
microSD card
│
├── bootfs
│    ├── boot files
│    └── configuration
│
└── rootfs
     └── Linux filesystem
```

For the U-Boot workflow, the built:

``` text
u-boot.bin
```

can be copied to the boot filesystem according to the target boot
configuration.

The board configuration can then be adjusted so that the Raspberry Pi
boot process uses U-Boot.

------------------------------------------------------------------------

# 32. UART and the Raspberry Pi Debug Probe

The U-Boot console can be accessed through UART.

A Debug Probe can provide the connection between the Raspberry Pi UART
and the development PC.

Conceptually:

``` text
Raspberry Pi 5
      │
      │ UART
      ▼
Debug Probe
      │
      │ USB
      ▼
Development PC
      │
      ▼
Serial Terminal
      │
      ▼
U-Boot Console
```

The Debug Probe does not run U-Boot.

Its role is to provide access to the board's debug/UART interface.

------------------------------------------------------------------------

# 33. U-Boot Console

Once U-Boot is running, the prompt is typically:

``` text
U-Boot>
```

The console lets you interact directly with the bootloader.

Important commands include:

  Command               Purpose
  --------------------- ---------------------------------------------------
  `help`                List available commands
  `help <command>`      Show help for a command
  `printenv`            Display environment
  `setenv NAME VALUE`   Change an environment variable
  `saveenv`             Save environment to configured persistent storage
  `boot`                Start the boot process

------------------------------------------------------------------------

# 34. QEMU: Running U-Boot Without Physical Hardware

After understanding U-Boot itself, QEMU provides a useful way to test
U-Boot on a development PC.

QEMU provides **virtual/simulated hardware**.

The important relationship is:

``` text
Development PC
      │
      ▼
    QEMU
      │
      ▼
Virtual ARM64 Machine
      │
      ▼
    U-Boot
```

QEMU is therefore a **testing environment for U-Boot**, not the
definition of U-Boot.

------------------------------------------------------------------------

# 35. Configure U-Boot for QEMU ARM64

Use:

``` bash
make qemu_arm64_defconfig
```

This uses:

``` text
configs/qemu_arm64_defconfig
```

and generates:

``` text
.config
```

Then build:

``` bash
make
```

The flow is:

``` text
configs/qemu_arm64_defconfig
          │
          ▼
        .config
          │
          ▼
         make
          │
          ▼
      u-boot.bin
```

------------------------------------------------------------------------

# 36. Launch U-Boot in QEMU

A typical command is:

``` bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a76 \
    -bios u-boot.bin \
    -m 2G \
    -nographic
```

This starts an ARM64 QEMU virtual machine and uses U-Boot as its boot
firmware.

------------------------------------------------------------------------

# 37. QEMU Command-Line Arguments

  Option               Meaning
  -------------------- --------------------------------------------------
  `-M virt`            Use QEMU's generic virtual ARM machine
  `-cpu cortex-a76`    Emulate a Cortex-A76 CPU
  `-bios u-boot.bin`   Use `u-boot.bin` as boot firmware
  `-m 2G`              Give the virtual machine 2 GB RAM
  `-nographic`         Use terminal/console instead of graphical output

------------------------------------------------------------------------

## 37.1 `-M virt`

``` bash
-M virt
```

selects QEMU's generic virtual ARM machine.

``` text
-M virt
   │
   ▼
Virtual ARM board
```

------------------------------------------------------------------------

## 37.2 `-cpu cortex-a76`

``` bash
-cpu cortex-a76
```

tells QEMU to emulate a Cortex-A76 CPU.

------------------------------------------------------------------------

## 37.3 `-bios u-boot.bin`

``` bash
-bios u-boot.bin
```

tells QEMU to use the previously built U-Boot binary as boot firmware.

The startup flow is:

``` text
QEMU starts
     │
     ▼
Loads u-boot.bin
     │
     ▼
U-Boot starts
     │
     ▼
U-Boot console
```

------------------------------------------------------------------------

## 37.4 `-m 2G`

``` bash
-m 2G
```

allocates 2 GB of virtual RAM.

------------------------------------------------------------------------

## 37.5 `-nographic`

``` bash
-nographic
```

uses the terminal instead of a graphical display.

``` text
QEMU
  │
  ▼
Terminal
  │
  ▼
U-Boot Console
```

------------------------------------------------------------------------

# 38. QEMU Startup Flow

``` text
Development PC
      │
      ▼
qemu-system-aarch64
      │
      ├── -M virt
      ├── -cpu cortex-a76
      ├── -bios u-boot.bin
      ├── -m 2G
      └── -nographic
      │
      ▼
Virtual ARM64 Machine
      │
      ▼
    U-Boot
      │
      ▼
U-Boot Console
```

------------------------------------------------------------------------

# 39. Exiting QEMU

When QEMU is running with:

``` text
-nographic
```

the terminal is attached to QEMU.

The exit sequence is:

``` text
Ctrl+A
   │
   ▼
Press X
   │
   ▼
QEMU exits
```

------------------------------------------------------------------------

# 40. QEMU Environment Persistence

If the QEMU setup does not provide appropriate persistent storage for
the U-Boot environment, changes may disappear after restarting QEMU.

For example:

``` text
setenv test hello
```

may work during the current session:

``` text
printenv test
```

showing:

``` text
test=hello
```

but after restarting QEMU, the variable may no longer exist.

The general rule is:

``` text
Persistence requires configured persistent storage.
```

This is the same U-Boot concept discussed earlier for physical hardware.

------------------------------------------------------------------------

# 41. Common Mistakes

## Mistake 1 --- Confusing Kconfig and Device Tree

Kconfig is primarily for software/build configuration.

Device Tree describes hardware.

``` text
Kconfig       → software configuration
Device Tree   → hardware description
```

------------------------------------------------------------------------

## Mistake 2 --- Thinking `setenv` Is Automatically Permanent

``` bash
setenv bootdelay 5
```

changes the current environment.

Persistence requires configured storage and saving the environment.

------------------------------------------------------------------------

## Mistake 3 --- Forgetting `saveenv`

If persistent storage is configured and you want a change to survive
reboot, the environment generally needs to be saved.

``` bash
setenv NAME VALUE
saveenv
```

------------------------------------------------------------------------

## Mistake 4 --- Assuming `saveenv` Always Works

`saveenv` depends on the configured environment backend/storage.

Do not assume that every target stores the environment in the same
place.

------------------------------------------------------------------------

## Mistake 5 --- Creating a Function Without Registering the Command

Defining:

``` c
do_packt()
```

does not automatically create a command named `packt`.

You also need:

``` c
U_BOOT_CMD(...)
```

------------------------------------------------------------------------

## Mistake 6 --- Forgetting `cmd/Makefile`

Creating:

``` text
cmd/packt.c
```

is not enough if the build system does not compile it.

------------------------------------------------------------------------

## Mistake 7 --- Forgetting to Rebuild

After modifying U-Boot:

``` bash
make
```

must be run to produce the updated binary.

------------------------------------------------------------------------

## Mistake 8 --- Confusing UCLASS and Driver

Remember:

``` text
UCLASS
   │
   └── Groups similar drivers

Driver
   │
   └── Hardware-specific implementation

Device
   │
   └── Actual device instance
```

------------------------------------------------------------------------

# 42. Important Concepts to Memorize

## U-Boot

``` text
U-Boot = bootloader
```

It prepares the system for booting an operating system and provides a
development/debugging interface.

------------------------------------------------------------------------

## Kconfig

``` text
Kconfig → software/build configuration
```

------------------------------------------------------------------------

## Defconfig

``` text
defconfig → predefined target configuration
```

------------------------------------------------------------------------

## `.config`

``` text
.config → full configuration used by the build
```

------------------------------------------------------------------------

## Driver Model

``` text
DM → framework for managing devices and drivers
```

------------------------------------------------------------------------

## UCLASS

``` text
UCLASS → category/interface for similar drivers
```

------------------------------------------------------------------------

## Device Tree

``` text
Device Tree → hardware description
```

------------------------------------------------------------------------

## Environment

``` text
Environment → configurable U-Boot state/settings
```

------------------------------------------------------------------------

## `bootcmd`

``` text
bootcmd → commands used for normal boot
```

------------------------------------------------------------------------

## `bootargs`

``` text
bootargs → kernel command-line arguments
```

------------------------------------------------------------------------

## `printenv`

``` bash
printenv
```

Shows environment variables.

------------------------------------------------------------------------

## `setenv`

``` bash
setenv NAME VALUE
```

Changes an environment variable.

------------------------------------------------------------------------

## `saveenv`

``` bash
saveenv
```

Attempts to save the environment to configured persistent storage.

------------------------------------------------------------------------

## `boot`

``` bash
boot
```

Starts the U-Boot boot process, commonly using `bootcmd`.

------------------------------------------------------------------------

## `U_BOOT_CMD()`

``` c
U_BOOT_CMD(...)
```

Registers a U-Boot command.

------------------------------------------------------------------------

## `do_packt()`

Contains the implementation of the custom `packt` command.

------------------------------------------------------------------------

## `cmd/Makefile`

Makes sure the new command source is included in the build.

------------------------------------------------------------------------

## QEMU

``` text
QEMU = virtual/simulated hardware
```

It can be used to run/test U-Boot without physical hardware.

------------------------------------------------------------------------

# 43. Command Cheat Sheet

  Command               Purpose
  --------------------- ---------------------------------------------------
  `help`                List U-Boot commands
  `help <command>`      Show command-specific help
  `printenv`            Display environment
  `setenv NAME VALUE`   Change an environment variable
  `saveenv`             Save environment to configured persistent storage
  `boot`                Start the boot process

Examples:

``` bash
help
help boot
printenv
setenv bootdelay 5
saveenv
boot
```

------------------------------------------------------------------------

# 44. Build Cheat Sheet

### Raspberry Pi 5

``` bash
make rpi_arm64_defconfig
make
```

### QEMU ARM64

``` bash
make qemu_arm64_defconfig
make
```

### Cross compiler

``` bash
export CROSS_COMPILE=aarch64-none-linux-gnu-
```

Basic build flow:

``` text
Target defconfig
      │
      ▼
    .config
      │
      ▼
     make
      │
      ▼
 u-boot.bin
```

------------------------------------------------------------------------

# 45. Custom Command Cheat Sheet

### Source

``` text
cmd/packt.c
```

### Function

``` c
static int do_packt(struct cmd_tbl *cmdtp, int flag,
                    int argc, char *const argv[])
{
    printf("Hello Packt World\n");
    return 0;
}
```

### Registration

``` c
U_BOOT_CMD(
    packt, 1, 1, do_packt,
    "Simple Packt command",
    ""
);
```

### Build-system entry

``` text
packt.o
```

inside:

``` text
cmd/Makefile
```

### Build

``` bash
make
```

### Test

``` text
U-Boot> help packt
U-Boot> packt
```

Expected:

``` text
Hello Packt World
```

------------------------------------------------------------------------

# 46. One-Minute Revision

If you only have one minute before a quiz or interview, remember:

``` text
                    U-BOOT
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
   Driver Model     Commands        Environment
       │               │                │
       ▼               ▼                ▼
    UCLASS          U_BOOT_CMD        bootcmd
    Driver          cmd_tbl           bootargs
    Device          cmd/              bootdelay
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                 Boot Process
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        Real Hardware           QEMU
        Raspberry Pi 5       Virtual ARM64
```

For a custom command:

``` text
cmd/packt.c
     │
     ▼
do_packt()
     │
     ▼
U_BOOT_CMD()
     │
     ▼
cmd/Makefile
     │
     ▼
make
     │
     ▼
u-boot.bin
     │
     ▼
U-Boot
     │
     ▼
packt
     │
     ▼
Hello Packt World
```

------------------------------------------------------------------------

# 47. Self-Test Questions

## Q1. What is U-Boot?

**Answer:** A bootloader used in many embedded systems to initialize the
system, load the operating system, and provide bootloader functionality
and a console.

------------------------------------------------------------------------

## Q2. What is the difference between Kconfig and Device Tree?

**Answer:**

``` text
Kconfig      → software/build configuration
Device Tree  → hardware description
```

------------------------------------------------------------------------

## Q3. What is a defconfig?

**Answer:** A predefined configuration containing default settings for a
target board/platform.

------------------------------------------------------------------------

## Q4. What does `make rpi_arm64_defconfig` do?

**Answer:** Selects the Raspberry Pi ARM64 default configuration and
generates the full `.config`.

------------------------------------------------------------------------

## Q5. What is cross-compilation?

**Answer:** Building software on one architecture for execution on
another architecture.

------------------------------------------------------------------------

## Q6. What is UCLASS?

**Answer:** A category/interface that groups similar U-Boot drivers.

------------------------------------------------------------------------

## Q7. What is a driver?

**Answer:** Hardware-specific software that implements the interface
needed to communicate with a device.

------------------------------------------------------------------------

## Q8. What is Device Tree used for?

**Answer:** Describing hardware to software, including information used
to identify and configure devices.

------------------------------------------------------------------------

## Q9. What does `U_BOOT_DRIVER()` do?

**Answer:** Registers a U-Boot driver with the Driver Model.

------------------------------------------------------------------------

## Q10. What does `U_BOOT_CMD()` do?

**Answer:** Registers a command and associates its command name with the
function that implements it.

------------------------------------------------------------------------

## Q11. What does `printenv` do?

**Answer:** Displays U-Boot environment variables.

------------------------------------------------------------------------

## Q12. What does `setenv` do?

**Answer:** Changes an environment variable in the current environment.

------------------------------------------------------------------------

## Q13. What does `saveenv` do?

**Answer:** Attempts to save the current environment to configured
persistent storage.

------------------------------------------------------------------------

## Q14. What is `bootcmd`?

**Answer:** An environment variable containing commands used for the
normal U-Boot boot process.

------------------------------------------------------------------------

## Q15. What is `bootargs`?

**Answer:** Kernel command-line arguments passed to Linux.

------------------------------------------------------------------------

## Q16. What does the `boot` command do?

**Answer:** Starts the U-Boot boot process, commonly using `bootcmd`.

------------------------------------------------------------------------

## Q17. Where is the custom command source located?

**Answer:**

``` text
cmd/packt.c
```

------------------------------------------------------------------------

## Q18. What function implements the `packt` command?

**Answer:**

``` c
do_packt()
```

------------------------------------------------------------------------

## Q19. Why is `U_BOOT_CMD()` needed?

**Answer:** Because defining `do_packt()` alone does not register
`packt` as a U-Boot command.

------------------------------------------------------------------------

## Q20. Why modify `cmd/Makefile`?

**Answer:** So the build system compiles the new command source into
U-Boot.

------------------------------------------------------------------------

## Q21. What happens when you type:

``` text
U-Boot> packt
```

**Answer:** U-Boot finds the registered `packt` command, calls
`do_packt()`, and prints:

``` text
Hello Packt World
```

------------------------------------------------------------------------

## Q22. What is QEMU?

**Answer:** A virtual/emulated hardware environment that can be used to
run and test software such as U-Boot.

------------------------------------------------------------------------

## Q23. What does `-M virt` do?

**Answer:** Selects QEMU's generic virtual ARM machine.

------------------------------------------------------------------------

## Q24. What does `-cpu cortex-a76` do?

**Answer:** Tells QEMU to emulate a Cortex-A76 CPU.

------------------------------------------------------------------------

## Q25. What does `-bios u-boot.bin` do?

**Answer:** Tells QEMU to use `u-boot.bin` as its boot firmware.

------------------------------------------------------------------------

## Q26. What does `-m 2G` do?

**Answer:** Allocates 2 GB of virtual RAM.

------------------------------------------------------------------------

## Q27. What does `-nographic` do?

**Answer:** Uses terminal/console output instead of a graphical display.

------------------------------------------------------------------------

## Q28. Why might an environment change disappear after restarting QEMU?

**Answer:** Because persistence requires appropriately configured
persistent environment storage.

------------------------------------------------------------------------

# 48. Final U-Boot Mental Model

The entire topic can be remembered as several connected layers:

``` text
┌─────────────────────────────────────────────┐
│                 U-BOOT                      │
│                                             │
│  Bootloader + Console + Drivers + Commands  │
└──────────────────────┬──────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Drivers        Commands      Environment
        │              │              │
        ▼              ▼              ▼
     UCLASS        U_BOOT_CMD       bootcmd
     Device        cmd_tbl          bootargs
     Device Tree   cmd/             bootdelay
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                 Boot Process
                       │
              ┌────────┴────────┐
              ▼                 ▼
       Raspberry Pi 5          QEMU
       Real Hardware      Virtual Hardware
```

And the complete learning path is:

``` text
Understand U-Boot
       │
       ▼
Understand configuration
(Kconfig / defconfig / .config)
       │
       ▼
Build U-Boot
       │
       ▼
Understand Driver Model
(UCLASS / Driver / Device / DT)
       │
       ▼
Understand Environment
(bootcmd / bootargs / etc.)
       │
       ▼
Understand Commands
(U_BOOT_CMD / cmd_tbl)
       │
       ▼
Run on Raspberry Pi 5
       │
       ▼
Use QEMU for testing
       │
       ▼
Modify U-Boot
       │
       ▼
Create custom commands
```

## Core Takeaway

> **U-Boot is the main subject. Kconfig and defconfig control how it is
> built, the Driver Model manages hardware drivers and devices, Device
> Tree describes hardware, environment variables control configurable
> boot behavior, and U-Boot commands provide an interactive interface.
> QEMU is simply one environment in which U-Boot can be tested, while
> Raspberry Pi 5 is an example of real hardware on which U-Boot can
> run.**

## Final Custom-Command Mental Model

``` text
                 U-BOOT SOURCE
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
      cmd/packt.c               cmd/Makefile
          │                         │
          ▼                         │
      do_packt()                    │
          │                         │
          ▼                         │
     U_BOOT_CMD() ◄─────────────────┘
          │
          ▼
        make
          │
          ▼
     u-boot.bin
          │
       ┌──┴──────────────┐
       ▼                 ▼
 Raspberry Pi 5         QEMU
       │                 │
       └────────┬────────┘
                ▼
           U-Boot Console
                │
                ▼
         U-Boot> packt
                │
                ▼
       Hello Packt World
```
