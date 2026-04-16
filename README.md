# Buildroot Project Template for Raspberry Pi 4

This repository is a simple Buildroot-based project template for Raspberry Pi 4.
It is organized as a `BR2_EXTERNAL` tree so the board configuration, custom files,
and build output can be managed cleanly in a separate Git repository.
The overall layout of this project was designed with the Buildroot manual's
recommended project layout in mind, specifically
["9.1. Recommended directory structure"](https://buildroot.org/downloads/manual/manual.html#customize-dir-structure).

## Overview

- Target board: Raspberry Pi 4 (64-bit)
- Build system: Buildroot
- Kernel configuration: `bcm2711`
- Defconfig: `configs/raspberrypi4_64_defconfig`

## Project Layout

- `buildroot/`: Buildroot source tree
- `configs/`: board defconfig files
- `dl/`: downloaded source archives
- `output/`: build output directory

## Build

Run the following commands from the project root:

```bash
# Configure the Raspberry Pi 4 64-bit build
make raspberrypi4_64_defconfig

# Build the full image
make -C output/raspberrypi4_64
```

## Build Output

After the build completes, generated files are placed under:

```bash
output/raspberrypi4_64/images/
```

This directory typically contains the kernel, device tree blobs, root filesystem,
and the final image files required to boot the board.
