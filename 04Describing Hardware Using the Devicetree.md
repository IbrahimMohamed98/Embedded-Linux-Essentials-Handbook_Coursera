# Raspberry Pi 5 Device Tree and Hardware Interfaces 

## Table of Contents

### Part I — Device Tree foundations
1. [What Device Tree is for](#1-what-device-tree-is-for)
2. [DTS, DTSI, DTB, and DTBO](#2-dts-dtsi-dtb-and-dtbo)
3. [Device Tree syntax: nodes, properties, labels, and references](#3-device-tree-syntax-nodes-properties-labels-and-references)
4. [The compile and decompile workflow](#4-the-compile-and-decompile-workflow)
5. [How Linux matches a device to a driver](#5-how-linux-matches-a-device-to-a-driver)
6. [Raspberry Pi 5 Device Tree source hierarchy](#6-raspberry-pi-5-device-tree-source-hierarchy)
7. [Device Tree overlays](#7-device-tree-overlays)
8. [A GPIO power-button example](#8-a-gpio-power-button-example)

### Part II — Raspberry Pi 5 practical hardware lessons
9. [ADS1115 ADC over I²C](#9-ads1115-adc-over-i²c)
10. [MCP4725 DAC over I²C](#10-mcp4725-dac-over-i²c)
11. [Raspberry Pi 5 GPIO](#11-raspberry-pi-5-gpio)
12. [BME280 environmental sensor over SPI](#12-bme280-environmental-sensor-over-spi)
13. [How the full Linux hardware stack fits together](#13-how-the-full-linux-hardware-stack-fits-together)
14. [Commands cheat sheet](#14-commands-cheat-sheet)
15. [Key concepts to memorize](#15-key-concepts-to-memorize)
16. [Self-test questions and answers](#16-self-test-questions-and-answers)

---

# Part I — Device Tree foundations

## 1. What Device Tree is for

A **Device Tree (DT)** is a structured description of the hardware in a system. It tells the operating system which hardware exists, how devices are connected, which addresses or buses they use, and which configuration details a driver needs. It is especially useful when one Linux kernel must support many boards built around similar processors but wired differently.

Without a Device Tree, board-specific hardware information would have to be hard-coded into kernel drivers or platform code. With Device Tree, much of the board description is data kept separate from the driver implementation.

A useful separation is:

- **Device Tree:** describes the hardware and its configuration.
- **Kernel driver:** contains the software logic for operating a device.
- **Kernel subsystem:** provides a standard framework for a class of devices, such as I²C, SPI, GPIO, or IIO.
- **Userspace:** applications and tools use the interfaces the kernel exposes.

```text
Board hardware
     |
     v
Device Tree describes devices, buses, addresses, interrupts, GPIOs, etc.
     |
     v
Linux parses the Device Tree
     |
     v
Kernel creates device objects and searches for matching drivers
     |
     v
Driver probes and initializes the hardware
     |
     v
A subsystem/device interface becomes available to userspace
```

Device Tree does not itself implement the driver or directly read sensor data. It describes the device so that the kernel can instantiate and configure it correctly.

### 1.1 Common Device Tree file types

| Extension | Meaning | Typical role |
|---|---|---|
| `.dts` | Device Tree Source | Top-level source for a board or a complete DT description |
| `.dtsi` | Device Tree Source Include | Reusable definitions included by one or more `.dts` files |
| `.dtb` | Device Tree Blob | Compiled binary Device Tree loaded by the boot process/kernel |
| `.dtbo` | Device Tree Blob Overlay | Compiled fragment that modifies or extends a base Device Tree |

The `.dts` and `.dtsi` files are human-readable source. The `.dtb` and `.dtbo` files are compiled binary forms.

## 2. DTS, DTSI, DTB, and DTBO

### 2.1 Why use `.dtsi` files?

A `.dtsi` file lets several board descriptions share common hardware definitions. For example, a SoC include may describe the CPU, interrupt controller, memory-mapped peripherals, and buses. A board-specific `.dts` can then include that shared file and add or change the wiring and peripherals for a particular board.

Conceptually:

```dts
// board-example.dts
/dts-v1/;

#include "soc-common.dtsi"
#include "board-common.dtsi"

/ {
    model = "Example board";
    compatible = "vendor,example-board";
};
```

This is an illustrative source snippet. Because `#include` is a C-preprocessor directive, source using it normally needs preprocessing before it is passed to `dtc`.

### 2.2 Base Device Tree versus overlay

A base `.dtb` describes the main board configuration. An overlay (`.dtbo`) is used when an optional device or configuration should be added without maintaining a separate complete base tree for every combination.

Examples of overlay use include enabling a peripheral on a bus, adding an I²C sensor, or changing pin configuration. An overlay can only work if its target labels and properties match the base Device Tree used on that system.

## 3. Device Tree syntax: nodes, properties, labels, and references

A Device Tree is a hierarchy of **nodes**. Each node can contain **properties** and child nodes.

```dts
/ {
    model = "Example board";
    compatible = "vendor,example-board";

    example_bus {
        status = "okay";

        example_device@20 {
            compatible = "vendor,example-device";
            reg = <0x20>;
        };
    };
};
```

- `/` is the root node.
- `example_bus` is a child node.
- `example_device@20` is a device node; the `@20` unit address usually corresponds to the address represented by `reg`, according to the bus's addressing rules.
- `compatible` is a string (or string list) used to identify the device type and match it with a driver.
- `reg` describes the device's address or resource on its parent bus. Its exact interpretation depends on the parent bus's address/size-cell settings.
- `status = "okay"` conventionally marks a device as enabled; `status = "disabled"` marks it disabled.

### 3.1 Labels and `&label` references

A label gives a node a source-level name so that another part of the Device Tree can refer to it. Example:

```dts
rp1_i2c1: i2c@...
```

Here `rp1_i2c1` is a label. Another fragment can refer to the node with `&rp1_i2c1`:

```dts
&rp1_i2c1 {
    status = "okay";
};
```

This is an **override/extension** of an existing node, not the creation of an unrelated node named `rp1_i2c1`. The exact label must exist in the Device Tree source used for the build; label availability can differ by kernel branch and board.

### 3.2 Common property forms

```dts
compatible = "vendor,device";  // string
status = "okay";               // string
reg = <0x62>;                   // cell/number array
interrupts = <...>;             // numeric cells; interpretation is binding-specific
```

The meaning of a property is determined by the binding for that device and bus. A syntactically valid property can still be semantically wrong if it does not match the binding or the hardware.

### 3.3 `#address-cells` and `#size-cells`

These properties tell child nodes how many cells represent addresses and sizes in `reg`. Do not assume every `reg` property is a single number: the format depends on the parent bus. For I²C devices, `reg` commonly contains the I²C slave address, while memory-mapped devices may use address/size pairs.

## 4. The compile and decompile workflow

The Device Tree compiler is called **`dtc`**. When a source file uses C preprocessor directives such as `#include`, a preprocessor step is normally needed first.

### 4.1 Source-to-binary pipeline

```text
.dts + included .dtsi files
          |
          | C preprocessor (cpp), when required
          v
preprocessed DTS source
          |
          | dtc
          v
.dtb (binary Device Tree)
```

For overlays, the output is commonly a `.dtbo` file. Overlay builds often use symbol/fixup options such as `-@`, depending on the build system and overlay requirements.

### 4.2 Compile a plain DTS file

```bash
dtc -I dts -O dtb -o rpi5.dtb rpi5.dts
```

Meaning:

- `-I dts`: input format is DTS text.
- `-O dtb`: output format is a binary DTB.
- `-o rpi5.dtb`: output filename.
- `rpi5.dts`: source file.

If the source uses `#include`, preprocess it first using the appropriate kernel/build-system include paths and macros. A generic example is:

```bash
cpp -nostdinc -I include -undef -x assembler-with-cpp rpi5.dts > rpi5.dts.preprocessed
dtc -I dts -O dtb -o rpi5.dtb rpi5.dts.preprocessed
```

This illustrates the two-stage workflow; real Linux kernel builds supply architecture-specific include paths and macros, so for kernel sources it is usually safer to use the kernel's build system.

### 4.3 Decompile a DTB into readable DTS

```bash
dtc -I dtb -O dts -o output.dts input.dtb
```

This is useful for inspection and debugging. The decompiled file may not look exactly like the original source: comments, include structure, labels, and formatting may not be recoverable from the binary.

### 4.4 Compile an overlay

A typical overlay compilation command is:

```bash
dtc -@ -I dts -O dtb -o example.dtbo example-overlay.dts
```

The `-@` option enables symbol information used by many overlay workflows. Exact options can vary with the build system and target.

## 5. How Linux matches a device to a driver

The central relationship is:

```text
Device Tree node
   | compatible = "vendor,device"
   v
Linux creates/identifies a device object
   |
   v
Driver match table contains a compatible entry
   |
   v
Driver probe() runs when the device and driver match
   |
   v
Driver initializes the hardware and registers with a subsystem
```

- **`compatible`:** connects the hardware description to a supported device type.
- **Driver match table:** lists compatible strings the driver supports.
- **`probe()`:** the driver's initialization callback. It typically obtains resources, checks the device, configures it, and registers it with a kernel subsystem.
- **Subsystem interface:** exposes the device in a standard way, such as IIO for an ADC/DAC, the input subsystem for a button, or SPI/I²C infrastructure for bus devices.

A Device Tree node existing does not guarantee that a driver is available or that probing succeeds. The driver may not be compiled into the kernel or installed as a module; the `compatible` may not match; the bus/device may be disabled; wiring, power, or address may be wrong; or `probe()` may fail.

### 5.1 Kernel built-in versus module

Kernel configuration symbols commonly distinguish:

- `=y`: build the driver into the kernel.
- `=m`: build it as a loadable module.
- unset: do not build it.

For example, `CONFIG_DAC_MCP4725=m` means the MCP4725 driver is built as a module. A module may need to be installed for the running kernel and loaded, either automatically or manually. A Device Tree node alone cannot replace a missing driver.

## 6. Raspberry Pi 5 Device Tree source hierarchy

The course material discussed the Raspberry Pi 5 board description as a hierarchy of files, including:

- `bcm2712-rpi-5-b.dts`: top-level board description for Raspberry Pi 5 Model B.
- `bcm2712.dtsi`: shared Broadcom BCM2712 SoC definitions.
- `bcm2712-rpi.dtsi`: Raspberry Pi-specific BCM2712 configuration shared by relevant boards.
- `rp1.dtsi` (or a similarly named RP1 include in the particular source tree): definitions for the RP1 I/O controller and its peripherals.

The exact filename and include hierarchy can vary by Linux kernel source version, so inspect the tree you are building rather than assuming every branch is identical.

The mental model is **SoC definitions + Raspberry Pi-specific configuration + board-specific details + optional overlays**. A top-level `.dts` includes shared `.dtsi` files, then can override nodes or add board-specific devices.

### 6.1 Following a peripheral node

When studying a device node, identify:

1. Which bus/controller contains it (I²C, SPI, GPIO, memory-mapped bus, etc.).
2. Its `compatible` string.
3. Its `reg` address/resource.
4. Whether the parent bus is enabled (`status`).
5. Any required interrupts, GPIOs, clocks, resets, regulators, or pinctrl configuration.
6. The kernel driver that matches the compatible string.
7. The userspace interface expected after successful probing.

## 7. Device Tree overlays

An overlay is a small Device Tree fragment intended to modify the base tree. A common structure is:

```dts
/dts-v1/;
/plugin/;

&some_existing_bus {
    status = "okay";

    sensor@48 {
        compatible = "vendor,sensor";
        reg = <0x48>;
    };
};
```

This is a pattern, not a ready-to-use overlay: `some_existing_bus`, `compatible`, and the address must match the actual board, hardware, and binding.

### 7.1 Typical overlay workflow on Raspberry Pi OS

1. Write the overlay source `.dts`.
2. Check that target labels and binding properties are correct for the target kernel Device Tree.
3. Compile to `.dtbo`, often using `dtc -@`.
4. Install the overlay in the boot partition's overlay directory, commonly `/boot/firmware/overlays/` on current Raspberry Pi OS installations.
5. Add a `dtoverlay=overlay-name` line to the appropriate boot configuration, commonly `/boot/firmware/config.txt`.
6. Reboot if required.
7. Verify the device was created and the driver probed successfully, using sysfs, subsystem tools, and `dmesg`.

Do not assume `/boot/firmware` is the correct path on every distribution or older Raspberry Pi OS installation. Confirm the active boot configuration.

### 7.2 Troubleshooting overlays

- **Compilation error:** inspect syntax, include paths, and `dtc` diagnostics.
- **Unknown label:** the label may not exist in the target base tree, or the wrong source/version is being used.
- **Device absent after reboot:** confirm the overlay name/path/configuration and whether the overlay was applied.
- **Device exists but no driver:** verify `compatible`, driver configuration/module installation, and kernel logs.
- **Probe fails:** verify supply, wiring, bus number, address, pinmux, interrupts, and required properties.

## 8. A GPIO power-button example

A common Linux pattern for a physical button is the `gpio-keys` binding, which describes a button connected to a GPIO line. A simplified illustrative node looks like this:

```dts
/ {
    gpio-keys {
        compatible = "gpio-keys";

        power-button {
            label = "Power Button";
            linux,code = <116>;
            gpios = <&some_gpio_controller 3 GPIO_ACTIVE_LOW>;
            debounce-interval = <10>;
        };
    };
};
```

This is an illustration, not a copy-and-paste Raspberry Pi 5 overlay: the GPIO controller label, flags, pin, and required headers must match the target Device Tree and wiring. `linux,code = <116>` corresponds to the Linux input key code `KEY_POWER` in the usual input-event definitions. `GPIO_ACTIVE_LOW` means the logical assertion is represented by a low electrical level.

The `gpio-keys` driver reads these properties, requests the GPIO, and reports button events through the Linux input subsystem. In driver code, helpers read properties from the firmware node/device tree; the driver then requests resources and sets up event reporting. The important concept is not memorizing a particular C helper call, but understanding that **the DT binding defines the data format and the driver consumes it**.

---

# Part II — Raspberry Pi 5 practical hardware lessons

## 8A. Overview of the practical lessons

These lessons demonstrate three ways software can interact with hardware on a Raspberry Pi 5:

| Hardware | Interface | Purpose | Linux software interface discussed |
|---|---|---|---|
| MCP4725 DAC | I²C | Converts a digital value into an analog output voltage | IIO subsystem and sysfs |
| GPIO pin | GPIO | Reads a digital input or controls a digital output | `pinctrl`, `gpioinfo`, legacy GPIO sysfs |
| BME280 sensor | SPI | Reads environmental sensor data; the first test reads its chip ID | SPI device interface through `spidev` |

A useful high-level model is:

```text
Application / user command
          |
          v
Linux userspace interface
          |
          v
Kernel subsystem and driver
          |
          v
Bus or GPIO controller
          |
          v
Physical hardware
```

The exact layers differ by device. For example, the BME280 chip-ID example uses Python and `spidev` to access SPI directly from userspace, while the MCP4725 example uses a kernel driver exposed through the IIO subsystem.

---

# 9. ADS1115 ADC over I²C

## 9.1 What an ADC does

An **analog-to-digital converter (ADC)** measures an analog input voltage and represents it as a digital number. The ADS1115 is a 16-bit ADC with an I²C interface. It can be configured for single-ended or differential measurements, subject to the chip's input and operating limits.

The course used an Adafruit ADS1115 breakout connected to a Raspberry Pi 5 over I²C. A key distinction: the sensor's Device Tree node is typically called `ads1115`, while Linux may identify the shared driver family as ADS1015 because the same kernel driver supports ADS1015 and ADS1115 devices. This does not necessarily mean the wrong chip was detected.

## 9.2 Driver configuration and Device Tree

The course material stated that `bcm2712_defconfig` already enables the driver as a kernel module for its setup. A configuration option ending in `=m` means the driver is built as a loadable module. The exact option and availability should be checked in the kernel version being used.

The Device Tree must describe the ADS1115 as a child of the correct I²C controller, with the correct compatible string and the actual address selected by the breakout's ADDR connection. Example structure only:

```dts
&some_i2c_controller {
    status = "okay";

    adc@48 {
        compatible = "ti,ads1115";
        reg = <0x48>;
        status = "okay";
    };
};
```

`some_i2c_controller` and `0x48` are examples. Use the real controller label and address for the board. Other addresses may be possible depending on the ADDR pin wiring.

## 9.3 IIO interface and verifying the device

The ADS1115 driver uses the Linux **Industrial I/O (IIO)** subsystem. After the device is successfully described, the driver is present, and probing succeeds, inspect IIO devices:

```bash
ls /sys/bus/iio/devices/
```

The device might appear as `/sys/bus/iio/devices/iio:device0/`, but never assume the index is always `0`; enumerate the directory and inspect its contents. For example:

```bash
for d in /sys/bus/iio/devices/iio:device*; do
    echo "== $d =="
    cat "$d/name" 2>/dev/null
    ls "$d"
done
```

The `name` file can show a driver-family name such as `ads1015`. The actual exposed attributes depend on the driver and kernel version. Raw ADC values must be interpreted using the channel configuration, scale, and driver documentation; they are not always direct voltage values without conversion.

## 9.4 ADS1115 troubleshooting checklist

1. Confirm I²C is enabled and the breakout is powered correctly.
2. Verify SDA/SCL wiring and common ground; use 3.3 V-compatible logic.
3. Confirm the Device Tree node is under the correct I²C controller and its address matches the hardware.
4. Confirm the driver is built/installed and that its compatible string matches.
5. Check `dmesg` for I²C or probe errors.
6. Inspect `/sys/bus/iio/devices/` rather than assuming `iio:device0`.
7. Check channel attributes and scale before converting raw readings into volts.

# 10. MCP4725 DAC over I²C

## 2.1 What a DAC Does

A **DAC** is a digital-to-analog converter. It takes a digital number and produces a corresponding analog voltage.

The **MCP4725** is a 12-bit DAC controlled over I²C.

Because it is 12-bit, it has:

```text
2^12 = 4096 possible codes
```

The code values are normally from `0` through `4095`.

For a 3.3 V reference/supply, an idealized explanation is:

| Digital code | Approximate output |
|---:|---:|
| `0` | Near 0 V |
| `2048` | About half the reference, around 1.65 V |
| `4095` | Near the reference voltage, around 3.3 V |

These are approximate values. The actual voltage depends on the board, supply/reference arrangement, device tolerances, and output loading.

**Mental model:** the digital code tells the DAC what voltage level to produce.

## 2.2 Wiring the MCP4725

The wiring discussed in the lesson is:

| MCP4725 connection | Raspberry Pi 5 physical pin | Purpose |
|---|---:|---|
| `VIN` | Pin 1 | 3.3 V supply |
| `GND` | Pin 6 | Ground |
| `SCL` | Pin 5 | I²C clock |
| `SDA` | Pin 3 | I²C data |
| `VOUT` | Multimeter positive probe | Measure the analog output |
| Multimeter ground | Pi ground, e.g. pin 6 | Common measurement reference |

I²C uses two signal lines:

- **SCL:** the clock line.
- **SDA:** the data line.

The DAC receives its digital control data over I²C and changes its analog output accordingly.

Before powering the circuit, confirm the pin labels on your exact breakout board. Do not assume that every breakout board uses the same physical layout.

## 2.3 Enabling the Kernel Driver

The kernel needs a driver that knows how to communicate with the MCP4725.

The relevant kernel configuration symbol discussed in the lesson is:

```text
CONFIG_DAC_MCP4725=m
```

The `=m` means **build the driver as a loadable kernel module**, rather than compiling it directly into the kernel image.

A typical workflow for a kernel source tree is:

```bash
make ARCH=arm64 bcm2712_defconfig
make ARCH=arm64 modules
```

A common module-installation pattern is:

```bash
INSTALL_MOD_PATH="$PWD/output" make ARCH=arm64 modules_install
```

These are examples, not a complete universal build recipe. The correct architecture, cross-compiler, kernel configuration, output directory, and installation process depend on your build environment. If you are using a course-provided kernel tree or prebuilt Raspberry Pi OS kernel, follow that environment's instructions.

**Remember the distinction:**

- Kernel configuration determines whether the driver is built.
- Device Tree describes the hardware instance and tells Linux where the device is connected.
- The driver performs the actual device communication.

Enabling the driver alone does not necessarily create a configured device.

## 2.4 Device Tree Overlay

A Device Tree overlay adds or modifies hardware description information without requiring you to rewrite the entire base Device Tree.

The MCP4725 is an I²C device, so the overlay must describe it under the correct I²C controller node.

Example structure:

```dts
/dts-v1/;
/plugin/;

&rp1_i2c1 {
    mcp4725@62 {
        compatible = "microchip,mcp4725";
        reg = <0x62>;
        vdd-supply = <&your_3v3_regulator>;
    };
};
```

### Understanding each part

| Line | Meaning |
|---|---|
| `/dts-v1/;` | Declares the Device Tree source format version |
| `/plugin/;` | Marks this as an overlay |
| `&rp1_i2c1` | References a labeled I²C controller node; the label must exist in the target Device Tree |
| `mcp4725@62` | Node name; `62` reflects the example I²C address |
| `compatible = "microchip,mcp4725";` | Identifies the device type so Linux can match a suitable driver |
| `reg = <0x62>;` | Specifies the I²C address `0x62` |
| `vdd-supply = <&your_3v3_regulator>;` | Example supply reference; this is a **placeholder**, not a guaranteed Raspberry Pi Device Tree label |

**Important:** replace `&your_3v3_regulator` with a valid regulator label from the Device Tree used by your system, or use the supply property expected by the binding for your kernel version. Do not copy a placeholder unchanged. Likewise, verify that `&rp1_i2c1` is the correct label and that the I²C bus is enabled for your specific setup.

The node name and the `compatible` property serve different purposes:

- The node name describes the node in the Device Tree.
- The `compatible` property is used for device/driver matching.
- The `reg` property gives the device address on its parent bus.

## 2.5 Compile and Install the Overlay

If the Device Tree Compiler (`dtc`) is installed, a typical compilation command is:

```bash
dtc -@ -I dts -O dtb -o add-mcp4725.dtbo add-mcp4725.dts
```

Meaning:

- `-@`: include symbol information useful for overlays.
- `-I dts`: input is Device Tree Source.
- `-O dtb`: output is a compiled Device Tree Blob.
- `-o add-mcp4725.dtbo`: name of the generated overlay file.

On a typical Raspberry Pi OS installation, an overlay may be installed under:

```text
/boot/firmware/overlays/
```

For example:

```bash
sudo cp add-mcp4725.dtbo /boot/firmware/overlays/
```

The configuration file is commonly:

```text
/boot/firmware/config.txt
```

Add an overlay entry such as:

```text
dtoverlay=add-mcp4725
```

Then reboot if required:

```bash
sudo reboot
```

The exact boot partition path can differ by OS version and installation. Verify the path on your device before copying files or editing configuration.

## 2.6 Testing the DAC Output

After reboot, check whether Linux has exposed an IIO device:

```bash
ls /sys/bus/iio/devices/
```

You may see a directory such as:

```text
iio:device0
```

Inspect its name and available attributes:

```bash
cat /sys/bus/iio/devices/iio:device0/name
ls /sys/bus/iio/devices/iio:device0/
```

The exact device number and filenames can differ. Do not assume that the MCP4725 will always be `iio:device0`; inspect the available devices first.

If the expected raw-output attribute is present, it may be named:

```text
out_voltage0_raw
```

The raw code represents the requested DAC output level. For example, a code around `2048` is approximately half scale when the reference is 3.3 V.

Measure `VOUT` with a multimeter, with the meter ground connected to the Raspberry Pi ground. A reported reading around 3.3 V for a near-full-scale code is consistent with the expected behavior of a 3.3 V setup, allowing for tolerances.

## 2.7 MCP4725 Troubleshooting

If the IIO device or expected output does not appear, check in this order:

1. **Wiring:** verify `VIN`, `GND`, `SDA`, and `SCL`.
2. **I²C bus:** make sure I²C is enabled and the device is connected to the intended bus.
3. **Address:** verify the board's actual I²C address; `0x62` is the example used here, not a guarantee for every board.
4. **Device Tree:** verify the controller label, node, `compatible`, address, and supply reference.
5. **Driver:** confirm the MCP4725 driver is built and available.
6. **Kernel logs:** inspect `dmesg` for I²C, DAC, or IIO-related errors.
7. **IIO path:** list the actual IIO devices instead of assuming the device index.
8. **Measurement:** confirm the multimeter ground shares a common ground with the Pi.

---

# 11. Raspberry Pi 5 GPIO

## 3.1 GPIO Basics

**GPIO** means General-Purpose Input/Output. A GPIO pin can commonly be configured as either an input or an output.

- **Input:** software reads the logic level supplied by the external circuit.
- **Output:** software sets the pin to a logic level.

For a typical Raspberry Pi GPIO signal:

| Logic level | Approximate voltage |
|---|---:|
| LOW (`0`) | Near 0 V |
| HIGH (`1`) | Near 3.3 V |

These are digital logic levels, not precision voltage measurements.

### Input mental model

```text
External circuit
      |
      v
GPIO configured as input
      |
      v
Linux reads logic level
      |
      +--> 0 / LOW
      |
      +--> 1 / HIGH
```

## 3.2 GPIO3 and Physical Pin 5

In the Raspberry Pi 40-pin header:

- **GPIO3** is physical header **pin 5**.
- Physical **pin 1** supplies 3.3 V.
- Physical **pin 6** is ground.

The lesson tests GPIO3 as an input. When configured as an input, connecting pin 5 to a known logic level can demonstrate how software sees LOW or HIGH.

**Electrical safety is essential:**

- Raspberry Pi GPIO is 3.3 V logic and is **not 5 V tolerant**.
- Never apply 5 V to a GPIO input.
- Do not connect a GPIO output driven LOW directly to 3.3 V; that can cause a short or damage.
- Follow the lab's wiring instructions and ensure the pin is configured appropriately before changing connections.

## 3.3 Using `pinctrl`

`pinctrl` is a Raspberry Pi-specific command-line utility for inspecting and configuring pin state.

To configure GPIO3 as an input:

```bash
pinctrl set 3 ip
```

Here:

- `3` identifies GPIO3.
- `ip` means input.

To inspect the pin:

```bash
pinctrl get 3
```

The output can include information about the pin's function and logic level. In the lesson's example:

- `lo` means LOW.
- `hi` means HIGH.
- `pu` indicates a pull-up setting.
- `pd` indicates a pull-down setting.
- `none` indicates no pull-up or pull-down is selected.

The exact output format can vary by tool version.

A line such as:

```text
no  pu | -- // PIN5/GPIO3 = none
```

is tool output describing pin state/configuration; interpret it using the `pinctrl` version and course context.

### Device Tree alias versus GPIO number

A Device Tree alias such as:

```dts
gpio3 = &pinctrl;
```

is a named reference to a labeled Device Tree node. It is **not automatically the same thing** as a Linux global GPIO number or a physical header pin number.

Keep these concepts separate:

| Concept | Example | What it identifies |
|---|---|---|
| Physical header pin | Pin 5 | Physical connector position |
| GPIO line offset | `3` | Line number within a GPIO chip, in the lesson's mapping |
| Device Tree label/alias | `&pinctrl`, `gpio3` | Reference to a Device Tree node |
| Legacy global GPIO number | `402` in the course's example | Old sysfs-style number derived from a particular chip base and offset |

## 3.4 Finding a GPIO Line with `gpioinfo`

Modern GPIO tools can help identify GPIO chips and lines.

For example:

```bash
gpioinfo | grep -B4 PIN5
```

In the course's environment, the output identified a chip called `gpiochip4`, with a line offset of `3` labeled `PIN5`.

A GPIO chip has its own set of line offsets. The line offset is not necessarily the same as a legacy global GPIO number.

The course's mapping used:

```text
Legacy GPIO base: 399
Line offset:        3
                  ---
Legacy GPIO number: 402
```

So, in that **specific example**:

```text
399 + 3 = 402
```

Do not assume that `gpiochip4`, base `399`, or global GPIO number `402` applies to every Raspberry Pi 5 setup. Chip numbering and global bases can vary with kernel and system configuration.

## 3.5 Legacy GPIO Sysfs

The course demonstrates the older GPIO sysfs interface.

First, go to the GPIO sysfs directory:

```bash
cd /sys/class/gpio
```

Export the example legacy GPIO number:

```bash
echo 402 > export
```

Enter the GPIO directory:

```bash
cd gpio402
```

Set it as an input:

```bash
echo in > direction
```

Read the input:

```bash
cat /sys/class/gpio/gpio402/value
```

The returned value means:

| Value | Meaning |
|---|---|
| `0` | LOW |
| `1` | HIGH |

This interface uses the legacy global GPIO number, so the example only works if your system uses the same mapping.

The standard GPIO sysfs interface is deprecated and may not be enabled in newer kernels. If `/sys/class/gpio/export` is missing or the example does not work, use the modern GPIO character-device tools supported by your system instead.

## 3.6 Modern GPIO Tools and Safety

The modern `libgpiod` command-line tools include:

- `gpioinfo`: lists GPIO chips and lines.
- `gpioget`: reads a GPIO line.
- `gpioset`: sets a GPIO line, subject to the tool version and line ownership semantics.

Use the chip and line identifiers reported by your own system rather than copying a global GPIO number from another machine.

Also note that `pinctrl` and GPIO line access are related but not identical:

- `pinctrl` can configure pin multiplexing and electrical settings.
- GPIO character-device tools request and operate GPIO lines.
- Legacy sysfs exposes GPIO operations through `/sys/class/gpio`, if supported.

---

# 12. BME280 environmental sensor over SPI

## 4.1 What the BME280 Measures

The **BME280** is an environmental sensor that can measure:

- Temperature
- Humidity
- Barometric pressure

The sensor supports I²C and SPI. This lesson uses **SPI**.

The first test does not yet calculate temperature, humidity, or pressure. It reads the sensor's chip-ID register to check whether SPI communication is working.

The expected chip ID is:

```text
0x60
```

## 4.2 SPI Fundamentals

**SPI** stands for Serial Peripheral Interface. It commonly uses four signals:

| Signal | Meaning | Typical direction from Raspberry Pi |
|---|---|---|
| `SCK` / `SCLK` | Serial clock | Pi provides the clock |
| `MOSI` | Master Out, Slave In | Pi sends data to sensor |
| `MISO` | Master In, Slave Out | Pi receives data from sensor |
| `CS` / `CE` | Chip Select | Pi selects the target device |

The Raspberry Pi is the SPI controller (traditionally called the master); the BME280 is the peripheral (traditionally called the slave).

### Why the breakout labels can be confusing

Some sensor boards label data pins from the sensor's perspective:

- `SDI` means data **into the sensor**, so connect it to Raspberry Pi **MOSI**.
- `SDO` means data **out of the sensor**, so connect it to Raspberry Pi **MISO**.

SPI transfers data in both directions at the same time. To receive a byte, the controller still has to generate clock pulses, commonly by sending a dummy byte.

## 4.3 Connecting the BME280

The lesson's wiring is:

| BME280 breakout pin | Raspberry Pi 5 physical pin | Purpose |
|---|---:|---|
| `VIN` | Pin 17 | 3.3 V supply |
| `GND` | Pin 20 | Ground |
| `SCK` | Pin 23 | SPI clock |
| `SDO` | Pin 21 | Pi MISO; data from sensor |
| `SDI` | Pin 19 | Pi MOSI; data to sensor |
| `CS` | Pin 24 | Chip select |

Check your exact breakout board's documentation and pin labels before powering it. This table reflects the wiring described in the lesson.

## 4.4 Enabling SPI

On Raspberry Pi OS, the course workflow enables SPI using:

1. Run:

   ```bash
   sudo raspi-config
   ```

2. Open **Interface Options**.
3. Select **SPI**.
4. Enable it.
5. Reboot if prompted.

After enabling SPI, inspect the device nodes:

```bash
ls /dev/spidev*
```

A common example is:

```text
/dev/spidev0.0
```

Your system may expose a different bus/device number, or no `spidev` device at all if the driver or Device Tree setup does not enable it.

## 4.5 Understanding `spidev`

`spidev` is a Linux userspace interface that lets an application perform SPI transfers without writing a dedicated kernel driver for every experiment.

A device path such as:

```text
/dev/spidev0.0
```

is conventionally interpreted as:

- `0`: SPI bus/controller number.
- `0`: device/chip-select number on that bus.

These numbers depend on the active hardware and Device Tree configuration.

The lesson discusses this kernel configuration:

```text
CONFIG_SPI_SPIDEV=m
```

The `=m` means `spidev` is built as a loadable kernel module.

However, building the module alone may not be enough. Linux must also have an appropriate SPI device configuration, typically through Device Tree, so the controller and peripheral are represented correctly.

## 4.6 Device Tree and the SPI Device

The course identifies the Raspberry Pi 5 base Device Tree source as:

```text
bcm2712-rpi-5-b.dts
```

The Device Tree describes hardware to Linux, including controllers and their attached devices.

For this lesson, the important chain is:

```text
Device Tree describes SPI controller and device
                 |
                 v
Linux creates/configures SPI device
                 |
                 v
spidev interface is available (if supported/configured)
                 |
                 v
Python opens /dev/spidevX.Y
                 |
                 v
SPI transfer reaches the sensor
```

Do not blindly add a generic `compatible = "spidev"` node without checking the kernel version and binding rules. Linux kernels may restrict generic `spidev` Device Tree matching. Use the device binding and configuration supported by the course's kernel and board setup.

## 4.7 Read the Chip ID with Python

The lesson uses Python's `spidev` module.

Example:

```python
import spidev

spi = spidev.SpiDev()
spi.open(0, 0)
spi.max_speed_hz = 1200000

reply = spi.xfer2([0xD0, 0x00])
print(reply)
print(hex(reply[1]))

spi.close()
```

### Line-by-line explanation

| Code | Meaning |
|---|---|
| `import spidev` | Imports the Python SPI interface |
| `spidev.SpiDev()` | Creates an SPI device object |
| `spi.open(0, 0)` | Opens bus 0, chip-select/device 0; this must match the system |
| `spi.max_speed_hz = 1200000` | Requests a 1.2 MHz SPI clock |
| `spi.xfer2([...])` | Performs an SPI transfer |
| `print(reply)` | Prints the returned bytes |
| `hex(reply[1])` | Displays the second returned byte in hexadecimal |
| `spi.close()` | Closes the SPI device |

The expected chip ID is `0x60`, which is decimal `96`.

If the response is represented as:

```python
[0, 96]
```

then:

```python
reply[1] == 96
```

and:

```python
hex(reply[1])
```

prints:

```text
0x60
```

The exact first returned byte is not the important check in this example; the expected chip-ID byte is the second byte, `reply[1]`.

## 4.8 Understanding the SPI Transfer

The transfer is:

```python
reply = spi.xfer2([0xD0, 0x00])
```

It sends two bytes:

1. `0xD0`: the BME280 chip-ID register address, with the read bit set in the SPI register-address convention used by the sensor.
2. `0x00`: a dummy byte. Its main purpose is to generate additional clock pulses so the sensor can return the register value.

SPI is full duplex: every byte sent by the controller is accompanied by a byte received from the peripheral. The byte received during the address phase is generally not the chip-ID value; the value is read during the following byte.

`xfer2()` commonly keeps chip select asserted across the transfer, which is useful for a register read. Exact transfer behavior depends on the Python binding and SPI controller driver.

### What does `0xD0` mean?

`0xD0` is hexadecimal. In decimal:

```text
0xD0 = 208
```

The chip-ID register address is `0xD0`; the expected value stored there is `0x60`.

Do not confuse:

- **Register address:** `0xD0`
- **Expected register value/chip ID:** `0x60`

The first tells the sensor which register to read. The second is the data the sensor should return.

## 4.9 BME280 Troubleshooting

If the chip ID is not `0x60`, check:

1. **Power and ground:** confirm the sensor has the correct supply and a shared ground with the Pi.
2. **Wiring:** verify `SCK`, `SDI` to MOSI, `SDO` to MISO, and `CS`.
3. **SPI enablement:** check that SPI is enabled and the system was rebooted if required.
4. **Device node:** inspect `ls /dev/spidev*`; confirm that the Python `open(bus, device)` values match the actual device node.
5. **Python package:** confirm the `spidev` module is installed in the Python environment being used.
6. **Permissions:** if opening the device returns a permission error, check the user's access to the SPI device.
7. **SPI mode and speed:** verify the mode expected by the sensor and try a lower clock speed if necessary.
8. **Device Tree/kernel setup:** make sure the SPI controller and target device are configured appropriately.
9. **Chip-select wiring:** verify the selected CS line matches the opened SPI device.
10. **Logs:** inspect `dmesg` for SPI/controller-related errors.

Reading the chip ID successfully is a useful first check that the wiring, device configuration, and basic SPI transaction are working. It does not by itself verify that temperature, humidity, and pressure conversion code is correct.

---

# 13. How the full Linux hardware stack fits together

The three examples use different interfaces, but the general idea is the same.

## MCP4725 DAC

```text
Application / test
       |
       v
IIO userspace attributes (sysfs)
       |
       v
MCP4725 kernel driver
       |
       v
I²C subsystem/controller
       |
       v
MCP4725 DAC
       |
       v
Analog output voltage
```

## GPIO input

```text
User command / application
       |
       v
pinctrl or GPIO userspace interface
       |
       v
GPIO controller/driver
       |
       v
Physical GPIO pin
       |
       v
LOW or HIGH logic level
```

## BME280 SPI test

```text
Python program
       |
       v
Python spidev module
       |
       v
/dev/spidevX.Y
       |
       v
Linux SPI subsystem/controller
       |
       v
BME280 sensor
       |
       v
Chip ID response (expected 0x60)
```

### Why Device Tree and drivers both matter

A useful rule is:

- **Device Tree:** describes which hardware exists and how it is connected/configured.
- **Kernel driver:** contains software that knows how to operate that hardware.
- **Kernel subsystem:** provides a common interface for a class of devices, such as IIO, GPIO, I²C, or SPI.
- **Userspace interface:** lets a program or user interact with the subsystem.

A driver can be compiled but fail to bind to a device if the hardware description or matching information is wrong. Conversely, a Device Tree node does not provide the device's functionality by itself; the appropriate driver or interface still needs to be available.

---

# 13A. Quick comparison table

| Feature | MCP4725 DAC | GPIO3 input | BME280 SPI |
|---|---|---|---|
| Main purpose | Produce an analog voltage | Read a digital logic level | Communicate with an environmental sensor |
| Hardware interface | I²C | GPIO | SPI |
| Signals | SDA, SCL | GPIO line | SCK, MOSI, MISO, CS |
| Kernel/userspace path discussed | MCP4725 driver + IIO | `pinctrl`, GPIO tools, legacy sysfs | SPI subsystem + `spidev` |
| Device Tree role | Describes DAC on I²C bus | Describes controller/pin configuration | Describes SPI controller/device configuration |
| Basic test | Set/read DAC code and measure VOUT | Read LOW/HIGH | Read chip ID register |
| Expected example result | Around 1.65 V at half scale with 3.3 V reference | `0` or `1` | `0x60` |

---

# 14. Commands cheat sheet

## Device Tree

```bash
# Compile a DTS source into a DTB when preprocessing is not needed
dtc -I dts -O dtb -o output.dtb input.dts

# Decompile a DTB into readable DTS
dtc -I dtb -O dts -o output.dts input.dtb

# Compile an overlay (common form)
dtc -@ -I dts -O dtb -o example.dtbo example-overlay.dts

# Search kernel messages for bus, driver, and probe errors
dmesg | grep -Ei 'i2c|spi|gpio|ads|mcp|bme|probe|firmware'
```

## MCP4725 / IIO

```bash
# List IIO devices
ls /sys/bus/iio/devices/

# Inspect a particular IIO device
cat /sys/bus/iio/devices/iio:device0/name
ls /sys/bus/iio/devices/iio:device0/

# Compile a Device Tree overlay
dtc -@ -I dts -O dtb -o add-mcp4725.dtbo add-mcp4725.dts

# Typical Raspberry Pi OS overlay location (verify on your system)
sudo cp add-mcp4725.dtbo /boot/firmware/overlays/
```

## GPIO

```bash
# Set GPIO3 as input
pinctrl set 3 ip

# Inspect GPIO3
pinctrl get 3

# Search gpioinfo output for a labeled line
gpioinfo | grep -B4 PIN5

# Legacy sysfs example only; requires the mapping to match your system
cd /sys/class/gpio
echo 402 > export
cd gpio402
echo in > direction
cat value
```

## SPI / BME280

```bash
# Enable SPI through the Raspberry Pi configuration tool
sudo raspi-config

# List SPI userspace device nodes
ls /dev/spidev*

# Inspect kernel log messages
dmesg | grep -i spi
```

## U-Boot/kernel build note

The course examples discussed building kernel modules and installing them to an output directory:

```bash
make ARCH=arm64 bcm2712_defconfig
make ARCH=arm64 modules
INSTALL_MOD_PATH="$PWD/output" make ARCH=arm64 modules_install
```

These commands are environment-dependent. They are included here as study references, not as a universal build recipe for every Raspberry Pi OS kernel tree.

---

# 15. Key concepts to memorize

1. **MCP4725 is a DAC:** a digital code controls an analog output voltage.
2. **MCP4725 uses I²C in this lesson:** `SDA` carries data and `SCL` carries the clock.
3. **A 12-bit DAC has 4096 codes:** values from `0` to `4095`.
4. **`CONFIG_DAC_MCP4725=m`:** build the DAC driver as a loadable module.
5. **Device Tree describes hardware; the driver operates it.**
6. **A Device Tree `compatible` string is used for matching a device with a suitable driver.**
7. **`reg = <0x62>` is an example I²C address**, not a universal address for every MCP4725 board.
8. **IIO is the Linux subsystem used by the DAC example** to expose the device's input/output attributes.
9. **GPIO3 maps to physical header pin 5** in the Raspberry Pi header pinout.
10. **`pinctrl set 3 ip` sets GPIO3 to input** in the lesson's command syntax.
11. **`gpioinfo` reports GPIO chips and line offsets.** A line offset is not necessarily a legacy global GPIO number.
12. **The `399 + 3 = 402` mapping is specific to the course's example.**
13. **Legacy GPIO sysfs is deprecated** and may not be available; modern GPIO character-device tools are preferred.
14. **Raspberry Pi GPIO uses 3.3 V logic and is not 5 V tolerant.**
15. **BME280 measures temperature, humidity, and pressure.**
16. **SPI uses clock, data in, data out, and chip select** in the common four-signal setup.
17. **SDI on the sensor connects to Pi MOSI; SDO connects to Pi MISO.**
18. **`CONFIG_SPI_SPIDEV=m` means build `spidev` as a module.**
19. **`/dev/spidev0.0` means SPI bus 0, device/chip-select 0** in the example naming convention.
20. **The BME280 chip-ID register address is `0xD0`; its expected value is `0x60`.**
21. **The dummy byte in an SPI read generates clocks** so the sensor can return data.
22. **A successful chip-ID read is a basic communication check**, not a complete sensor-measurement test.

---

# 16. Self-test questions and answers

Try answering before looking at the answers.

## Questions

1. What does a DAC do?
2. Why does a 12-bit DAC have 4096 possible codes?
3. What do the I²C signals SDA and SCL do?
4. What does `CONFIG_DAC_MCP4725=m` mean?
5. What is the difference between Kconfig and Device Tree?
6. What is the purpose of the `compatible` property in a Device Tree node?
7. In the example overlay, what does `reg = <0x62>` represent?
8. What is the purpose of the Linux IIO subsystem in the MCP4725 example?
9. Which physical header pin is GPIO3?
10. What does `pinctrl set 3 ip` do?
11. Why should you not assume that GPIO number `402` works on every Raspberry Pi?
12. Why is legacy GPIO sysfs not the preferred interface for new applications?
13. Why must 5 V not be applied to a Raspberry Pi GPIO input?
14. What measurements can the BME280 provide?
15. What are SCK, MOSI, MISO, and CS used for?
16. On a breakout board labeled SDI and SDO, which connects to Pi MOSI?
17. What does `CONFIG_SPI_SPIDEV=m` mean?
18. What does `/dev/spidev0.0` identify?
19. In `spi.xfer2([0xD0, 0x00])`, why is a second byte sent?
20. What is the difference between the BME280 register address `0xD0` and chip ID `0x60`?

## Answers

1. It converts a digital code into an analog output, such as a voltage.
2. Because `2^12 = 4096` distinct codes are available, from `0` to `4095`.
3. SDA carries I²C data; SCL carries the I²C clock.
4. The MCP4725 driver is built as a loadable kernel module.
5. Kconfig selects software/build features; Device Tree describes hardware and its configuration.
6. It identifies the device type for matching with a suitable driver.
7. It specifies the example device's I²C address.
8. It provides a standard Linux userspace interface for the DAC.
9. Physical header pin 5.
10. It configures GPIO3 as an input using the lesson's `pinctrl` syntax.
11. GPIO chip numbering and legacy global bases can differ by system and kernel setup.
12. The sysfs GPIO interface is deprecated and may be disabled; modern GPIO character-device tools are preferred.
13. Raspberry Pi GPIO is 3.3 V logic and is not 5 V tolerant.
14. Temperature, humidity, and barometric pressure.
15. SCK is clock, MOSI sends controller-to-peripheral data, MISO receives peripheral-to-controller data, and CS selects the peripheral.
16. SDI connects to Pi MOSI.
17. The `spidev` driver is built as a loadable module.
18. SPI bus 0, device/chip-select 0 in the example.
19. SPI needs clock pulses to receive the returned byte; sending the dummy byte generates those clocks.
20. `0xD0` selects the chip-ID register; `0x60` is the expected value stored in that register.

---

### Additional Device Tree and ADS1115 review

1. **What is the difference between DTS and DTB?** DTS is readable source; DTB is the compiled binary used by the boot/kernel flow.
2. **Why is preprocessing sometimes needed before `dtc`?** Because directives such as `#include` are handled by the C preprocessor, not directly by `dtc`.
3. **What does `&label` do in an overlay?** It references an existing labeled node so the fragment can extend or change it.
4. **Does a Device Tree node guarantee that a driver will work?** No. A matching driver must exist, be built/installed, match the `compatible`, and probe successfully.
5. **What does `CONFIG_DRIVER=m` mean?** The driver is built as a loadable kernel module.
6. **Why might an ADS1115 appear under a driver name containing ADS1015?** The Linux driver family supports both ADS1015 and ADS1115 variants.
7. **Should you assume the ADC appears as `iio:device0`?** No. Enumerate `/sys/bus/iio/devices/` and inspect each device's `name` and attributes.
8. **What should you verify when an overlay fails?** Target label, syntax/compile output, overlay path/configuration, `compatible`, driver availability, bus/address, wiring, and kernel logs.

# Final Mental Model

```text
                         RASPBERRY PI 5
                               |
       +-----------------------+------------------------+
       |              |                |               |
       v              v                v               v
   ADS1115 ADC   MCP4725 DAC         GPIO3          BME280
       |              |                |               |
       v              v                v               v
      I²C            I²C           Digital I/O         SPI
       |              |                |               |
       v              v                v               v
   Kernel ADC     Kernel DAC      GPIO/pinctrl      SPI + spidev
       |              |                |               |
       v              v                v               v
      IIO            IIO          LOW or HIGH      Python test
  digital reading  analog output                    chip ID 0x60
```

The central idea is that **the physical bus, Device Tree description, driver or kernel subsystem, and userspace interface must agree**. When a hardware test fails, check those layers one by one instead of assuming the problem is only in the application code.
