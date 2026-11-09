# Embedded Linux Image Building: Yocto Project and Buildroot

A focused study guide covering the introductory lessons on the **Yocto
Project** and **Buildroot**: their purpose, architecture, build process,
important terminology, differences, advantages, disadvantages, and how
to choose between them.

------------------------------------------------------------------------

## Table of Contents

1.  [Introduction: Building an Embedded Linux
    Image](#1-introduction-building-an-embedded-linux-image)
2.  [The Yocto Project](#2-the-yocto-project)
    -   [2.1 What is Yocto?](#21-what-is-yocto)
    -   [2.2 OpenEmbedded and BitBake](#22-openembedded-and-bitbake)
    -   [2.3 Recipes](#23-recipes)
    -   [2.4 Layers](#24-layers)
    -   [2.5 `.bbappend` Files](#25-bbappend-files)
    -   [2.6 How the Yocto Components Work
        Together](#26-how-the-yocto-components-work-together)
3.  [Buildroot](#3-buildroot)
    -   [3.1 What is Buildroot?](#31-what-is-buildroot)
    -   [3.2 Buildroot vs. Yocto: Basic
        Differences](#32-buildroot-vs-yocto-basic-differences)
    -   [3.3 Buildroot Build Process](#33-buildroot-build-process)
    -   [3.4 Toolchains](#34-toolchains)
    -   [3.5 Kconfig and
        `make menuconfig`](#35-kconfig-and-make-menuconfig)
    -   [3.6 Make and Makefiles](#36-make-and-makefiles)
4.  [Buildroot vs. Yocto: Advantages and
    Disadvantages](#4-buildroot-vs-yocto-advantages-and-disadvantages)
    -   [4.1 Buildroot Advantages](#41-buildroot-advantages)
    -   [4.2 Buildroot Disadvantage](#42-buildroot-disadvantage)
    -   [4.3 Yocto Advantages](#43-yocto-advantages)
    -   [4.4 Yocto Disadvantages](#44-yocto-disadvantages)
5.  [How to Choose Between Buildroot and
    Yocto](#5-how-to-choose-between-buildroot-and-yocto)
6.  [Quick Comparison](#6-quick-comparison)
7.  [Key Terms Cheat Sheet](#7-key-terms-cheat-sheet)
8.  [Final Mental Model](#8-final-mental-model)

------------------------------------------------------------------------

## 1. Introduction: Building an Embedded Linux Image

An embedded Linux system is made from several components that must work
together. A typical system includes:

-   **Bootloader:** Initializes the hardware and starts the Linux
    kernel.
-   **Linux kernel:** Manages the processor, memory, processes, and
    hardware.
-   **Device Tree:** Describes the hardware so the kernel can understand
    the board and its peripherals.
-   **Root filesystem (rootfs):** Contains user-space programs,
    libraries, scripts, and configuration files.
-   **Applications:** The software that performs the device's intended
    function.
-   **Toolchain:** The compiler and related tools used to build software
    for the target processor.

A build framework automates the process of obtaining, configuring,
compiling, and assembling these components into an image that can be
flashed onto the target device.

For example, a Raspberry Pi 5 image may contain the boot files, Linux
kernel, Device Tree, drivers, root filesystem, libraries, and
applications required by the project.

## 2. The Yocto Project

### 2.1 What is Yocto?

The **Yocto Project** is a framework and ecosystem for creating
customized embedded Linux distributions.

Rather than manually collecting and building every component, Yocto
provides a structured way to describe the software that should be built
and how it should be configured.

It can be used to create an image containing components such as:

-   Bootloader
-   Linux kernel
-   Device Tree
-   Drivers
-   Libraries
-   Applications
-   Root filesystem

### 2.2 OpenEmbedded and BitBake

Two important names in the Yocto build system are **OpenEmbedded** and
**BitBake**.

-   **Yocto Project:** The overall project, tools, and ecosystem for
    creating customized embedded Linux systems.
-   **OpenEmbedded:** Provides much of the build metadata and
    infrastructure used by the build system.
-   **BitBake:** The build engine. It reads metadata and executes the
    tasks needed to build software and images.

BitBake can perform tasks such as:

1.  Read recipes and configuration.
2.  Retrieve source code.
3.  Apply patches.
4.  Configure and compile software.
5.  Install files and create packages.
6.  Help assemble the final image.

**Remember:** OpenEmbedded provides the metadata and infrastructure;
BitBake processes that information and performs the build tasks.

### 2.3 Recipes

A **recipe** describes how BitBake should obtain, build, and install a
software component.

Recipes normally use the `.bb` file extension.

A recipe may specify:

-   Where to obtain the source code.
-   Which patches to apply.
-   How to configure and compile the software.
-   Which files to install and where to install them.

Not every recipe compiles a program. A recipe can also install scripts,
configuration files, or other resources.

For example, a recipe could describe how to install a sensor-monitoring
application into a Raspberry Pi image.

**Important:** Having a recipe available does not automatically mean
that its software will be included in the final image. The image
configuration and package selections determine what is installed.

### 2.4 Layers

A **layer** is a collection of related Yocto metadata. It can contain
recipes, configuration files, and recipe extensions.

Layers organize functionality and make it easier to reuse or customize
build information.

Examples:

-   `meta-raspberrypi`: Provides Raspberry Pi board support.
-   `meta-swupdate`: Provides metadata for integrating SWUpdate.

A layer commonly contains a `conf/layer.conf` file.

#### What is `conf/layer.conf`?

This file tells BitBake important information about the layer, including
how the layer is configured and where its metadata and recipes can be
found.

### 2.5 `.bbappend` Files

A `.bbappend` file extends or modifies an existing recipe.

It is useful when we need to customize a recipe without editing the
original file, especially when that recipe comes from a vendor or
third-party layer.

For example:

-   An existing recipe installs an application.
-   Our project needs to install an additional configuration file.
-   We can use a `.bbappend` file to add that customization while
    leaving the original recipe unchanged.

### 2.6 How the Yocto Components Work Together

The basic relationship is:

``` text
Yocto Project
    |
    +-- OpenEmbedded: build metadata and infrastructure
    |
    +-- Layers: organize recipes and configuration
    |      |
    |      +-- Recipes (.bb)
    |      +-- Recipe extensions (.bbappend)
    |      +-- Layer configuration (conf/layer.conf)
    |
    +-- BitBake: reads metadata and executes build tasks
             |
             +-- Retrieves source code
             +-- Applies patches
             +-- Configures and compiles software
             +-- Creates packages and images
```

The result is a customized embedded Linux image.

## 3. Buildroot

### 3.1 What is Buildroot?

**Buildroot** is a tool that automates the creation of embedded Linux
systems.

Like Yocto, it can retrieve source code, build the toolchain and
software, and assemble the components into a bootable image.

For a Raspberry Pi 5 project, Buildroot can help build the bootloader,
kernel, Device Tree, applications, libraries, and root filesystem.

### 3.2 Buildroot vs. Yocto: Basic Differences

The two frameworks have similar goals but use different build
mechanisms.

  -----------------------------------------------------------------------
  Feature                 Yocto                   Buildroot
  ----------------------- ----------------------- -----------------------
  Main build instructions Recipes (`.bb`)         Package Makefiles

  Build engine            BitBake                 Make

  Configuration           Yocto configuration and Kconfig
                          metadata                

  Software installation   Commonly creates        Generally installs
                          packages before image   files directly into the
                          assembly                target root filesystem

  Learning curve          Steeper                 Generally easier to
                                                  start with
  -----------------------------------------------------------------------

#### Packages versus direct installation

Yocto commonly creates software packages, such as:

-   `.rpm`
-   `.deb`
-   `.ipk`

These packages contain applications, scripts, libraries, or
configuration files that can be installed into the root filesystem.

Buildroot generally installs the built files directly into the target
root filesystem instead of relying on a separate package-generation and
installation workflow.

### 3.3 Buildroot Build Process

Buildroot follows a process similar to Yocto:

``` text
1. Prepare a toolchain
        |
        v
2. Select the applications and components
        |
        v
3. Retrieve their source code
        |
        v
4. Cross-compile the software
        |
        v
5. Install files into the target root filesystem
        |
        v
6. Generate the embedded Linux image
```

In simple terms, Buildroot determines what the target system needs,
builds those components for the target processor, and places the
resulting files in the correct locations.

### 3.4 Toolchains

A **toolchain** is a collection of tools used to build software for a
target processor. It typically includes a compiler, linker, assembler,
and supporting utilities.

For example, when building software for the Raspberry Pi 5, the compiler
must generate binaries compatible with its ARM64 processor.

Buildroot supports two approaches:

-   **Internal toolchain:** Buildroot downloads and builds the toolchain
    as part of its build process.
-   **External toolchain:** Buildroot uses a toolchain that was obtained
    or built separately.

An external toolchain can be useful when a project requires a particular
compiler or feature that is not available in the toolchain generated by
Buildroot, or when the required target architecture is not supported by
the internal toolchain.

### 3.5 Kconfig and `make menuconfig`

Buildroot uses **Kconfig**, the same configuration system used by the
Linux kernel.

Kconfig provides a menu interface for selecting and configuring
features.

In Buildroot, configuration options include:

-   Target architecture and processor.
-   Toolchain settings.
-   Linux kernel options.
-   Packages and applications.
-   Root filesystem settings.

After downloading and extracting a Buildroot release, enter its
directory and run:

``` bash
make menuconfig
```

This opens the Buildroot configuration menu.

The interface should look familiar if you have already used Linux kernel
`menuconfig`.

**Note:** The exact Buildroot version and download commands depend on
the release being used. The essential point in this lesson is that
`make menuconfig` opens the configuration interface.

### 3.6 Make and Makefiles

Buildroot uses **Make** as its fundamental build tool.

-   **Make:** A tool that reads build instructions and executes the
    required commands.
-   **Makefile:** A file containing those instructions.

A Makefile can describe:

-   Which source files to compile.
-   Which compiler and options to use.
-   Which dependencies are required.
-   How and where to install the application.

A Buildroot package has build instructions that tell Make how to prepare
and install that package.

The course's `elehbr` example is a simple application that prints
`Hello World`. Its purpose is to introduce the kind of Makefile used by
Buildroot. The detailed Makefile syntax can be learned separately.

## 4. Buildroot vs. Yocto: Advantages and Disadvantages

### 4.1 Buildroot Advantages

1.  **Easier to learn:** Buildroot uses Makefiles and Kconfig, which are
    familiar to many embedded Linux developers.
2.  **Active community:** It has contributors and maintainers who
    develop and support the project.
3.  **Hardware support:** It supports a wide range of boards and CPU
    architectures.
4.  **Potentially smaller images:** It is straightforward to include
    only the components needed by the target system.
5.  **Usually faster initial builds:** A first build is typically faster
    than a comparable first Yocto build.

### 4.2 Buildroot Disadvantage

Buildroot's simplicity can mean less sophisticated handling of changes
and dependencies.

After some configuration or source-code changes, it may not rebuild
every component that needs rebuilding. Depending on the change, the
developer may need to clean and rebuild affected packages or perform a
more complete build.

The practical lesson is to understand Buildroot's rebuild and clean
targets and use the appropriate one when changing package sources or
configuration.

### 4.3 Yocto Advantages

1.  **Broad hardware support:** Many board and silicon vendors provide
    Yocto support.
2.  **Layered customization:** Layers allow projects to add or change
    functionality without directly modifying the original vendor
    metadata.
3.  **Large ecosystem:** Third-party layers can provide applications,
    services, frameworks, and update mechanisms.
4.  **Integrated build infrastructure:** The Yocto Project provides an
    established build environment using OpenEmbedded and BitBake.
5.  **Efficient subsequent builds:** The shared state cache can reuse
    results from previous build tasks.

#### What is the Shared State Cache (sstate)?

The **shared state cache**, often called the **sstate cache**, stores
reusable results from completed build tasks.

For example:

-   During the first build, Yocto compiles a component.
-   During a later build, if the inputs for that task have not changed,
    Yocto may reuse the cached result instead of compiling it again.

This can make later builds much faster.

### 4.4 Yocto Disadvantages

1.  **Larger default images:** A default Yocto image may be larger than
    a minimal Buildroot image. It can be reduced through customization.
2.  **Higher resource requirements:** Building Yocto from scratch
    generally needs more CPU, memory, and disk space.
3.  **More disk usage:** Source downloads, build output, packages, and
    cached data can consume substantial storage.
4.  **Steeper learning curve:** Developers need to understand concepts
    such as BitBake, recipes, layers, and configuration.

## 5. How to Choose Between Buildroot and Yocto

There is no single framework that is best for every project. The choice
depends on the hardware, project requirements, team experience, and
available development resources.

### Choose Buildroot when:

-   You need a small and relatively simple embedded Linux system.
-   You want to get started quickly.
-   Your team is new to embedded Linux build frameworks.
-   Your software requirements are relatively straightforward.
-   Development PC resources are limited.

The course gives a root filesystem smaller than approximately **8 MB**
as an example where Buildroot may be a suitable choice. This is a
guideline from the lesson, not a strict technical limit.

### Choose Yocto when:

-   The product needs many software components or frameworks.
-   You need extensive customization.
-   You want to reuse existing third-party layers.
-   The project benefits from a structured, reusable build
    configuration.
-   You need the package and layer ecosystem provided by Yocto.

### The Most Important Rule: Check the Hardware Vendor's BSP

Before choosing a framework, check what your hardware vendor provides.

**BSP** stands for **Board Support Package**. It is the collection of
software and configuration needed to support a particular board or
processor.

A BSP may include:

-   Bootloader support and configuration.
-   Linux kernel sources or patches.
-   Device Tree files.
-   Hardware drivers.
-   Board-specific configuration.
-   Build instructions and integration files.

For example, if a board vendor provides a complete Yocto BSP, using
Yocto is often the practical choice. Replacing it with Buildroot may
require significant additional engineering work.

Likewise, if the vendor provides a Buildroot BSP, starting with
Buildroot may save time.

If the vendor supports both frameworks, or does not prescribe one,
compare the project requirements and team experience.

## 6. Quick Comparison

  -----------------------------------------------------------------------
  Consideration           Buildroot               Yocto
  ----------------------- ----------------------- -----------------------
  Getting started         Easier                  More involved

  Initial build           Typically faster        Typically slower

  Later builds            Depends on rebuilds and Can benefit greatly
                          package changes         from sstate

  Minimal image           Often easier to achieve Possible, but may need
                                                  more customization

  Customization model     Makefiles and Kconfig   Layers, recipes, and
                                                  BitBake

  Third-party ecosystem   Available               Broad layer ecosystem

  Build resource needs    Generally lower         Generally higher

  Best fit                Smaller or simpler      Complex, highly
                          systems                 customized products
  -----------------------------------------------------------------------

These are general tendencies. Actual build times, image sizes, and
resource requirements depend on the configuration, hardware, software
selection, and build environment.

## 7. Key Terms Cheat Sheet

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  Embedded Linux image                The files and components needed to
                                      boot and run Linux on a target
                                      device

  Yocto Project                       Framework and ecosystem for
                                      creating customized embedded Linux
                                      distributions

  OpenEmbedded                        Build metadata and infrastructure
                                      used by Yocto

  BitBake                             Build engine that processes
                                      metadata and runs build tasks

  Recipe (`.bb`)                      Instructions for obtaining,
                                      building, and installing a software
                                      component

  Layer                               Collection of related Yocto
                                      metadata

  `.bbappend`                         File used to extend or modify an
                                      existing recipe

  `layer.conf`                        Configuration file for a Yocto
                                      layer

  Buildroot                           Tool for building embedded Linux
                                      systems

  Make                                Tool that executes build
                                      instructions

  Makefile                            File containing instructions used
                                      by Make

  Kconfig                             Configuration system used by the
                                      Linux kernel and Buildroot

  Toolchain                           Compiler and related tools for
                                      building software

  Cross-compilation                   Building software on one machine
                                      for a different target architecture

  Root filesystem                     User-space files, applications,
                                      libraries, and configuration used
                                      by Linux

  Artifact/package                    Packaged software output, such as
                                      `.rpm`, `.deb`, or `.ipk`

  sstate cache                        Yocto cache of reusable build-task
                                      results

  BSP                                 Board Support Package: software and
                                      configuration for a hardware
                                      platform
  -----------------------------------------------------------------------

## 8. Final Mental Model

Think of both frameworks as two different ways to achieve the same goal:
building a complete embedded Linux image.

``` text
                  Embedded Linux Image
                           ^
                           |
             +-------------+-------------+
             |                           |
          Buildroot                     Yocto
             |                           |
       Make + Kconfig             BitBake + OpenEmbedded
             |                           |
       Package Makefiles           Layers + Recipes
             |                           |
       Build components             Build components
             |                           |
       Install into rootfs          Create packages and
             |                      assemble the image
             +-------------+-------------+
                           |
                     Bootable image
```

The most important points to remember:

-   **Yocto:** Uses BitBake, OpenEmbedded, layers, and recipes. It
    provides extensive customization and a broad ecosystem.
-   **Buildroot:** Uses Make, Makefiles, and Kconfig. It is generally
    simpler and well suited to smaller or less complex systems.
-   **BSP first:** Prefer the framework supported by the hardware vendor
    unless there is a strong project-specific reason to choose
    otherwise.
