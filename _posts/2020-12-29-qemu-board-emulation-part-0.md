---
layout: post
title: QEMU Board Emulation - Introduction
tags: linux qemu u-boot
---

There are many cheap COTS (Commercial Off-The-Shelf) development boards available for playing with and learning Embedded linux: [Raspberry Pi](https://www.raspberrypi.org/), [BeagleBone](https://beagleboard.org/bone), [OrangePi](http://www.orangepi.org/) ...

The benefit of using COTS platform is that there are **A LOT** of available resources and examples, so one can create (mostly) functional prototype in a very short time. And this is good for someone who needs a quick solution for some problem without the need to go go into all the details.

However, for someone who wishes to go deeper into Embedded linux, boot process, optimization, driver development, etc, the COTS platform can provide only limited experience. Yes, one can develop drivers for some devices connected over external serial busses (even though someone has probably already developed a driver for that device), but that is about everything that can be done. There is no playing with memory maps, system-level device tree, bootloader adjustment, i.e. all of the things that are done when a custom board needs to be brought up.

One solution is to use development board based on <abbr title="System on Chip">SoC</abbr> which combines Hard CPU core and FPGA parts, like [Xilinx Zynq](https://www.xilinx.com/products/silicon-devices/soc/zynq-7000.html) or [Altera Cyclone](https://www.intel.com/content/www/us/en/products/programmable/soc/cyclone-v.html). This SoC allows custom memory mapped hardware to be developed, which would require the developer to create a novel driver and learn a lot in the process. However, the 'creative' process is limited only to certain devices, not the system as a whole.

<br />

This is where [QEMU](https://www.qemu.org/) (Quick EMUlator) can be very useful. QEMU has been used in industry during the design stage for new SoCs, since software development can be done before the hardware is not production ready (ZynqMP, RiscV).

QEMU has support for various development boards and devices, but new devices and development boards can be created with ease. For instance, one can create a new platform or I2C device, new system on chip with completely new memory map, or even new architecture (ok, highly unlikely that someone will go this far for educational purposes, but it is possible).

<br />

These series of posts will introduce QEMU board emulation for ARM architecture. The idea is to go from bootloader (U-Boot and Barebox), over kernel and root file-system (rootfs) creation. The first step will be to go over the whole procedure using several available boards:

* ARM Versatile Express A9
* <del>OrangePI PC</del> - covered in detail in [QEMU docs](https://qemu.readthedocs.io/en/latest/system/arm/orangepi.html)
* <del>IMX6 SabreLite</del> - covered in detail in [QEMU docs](https://qemu.readthedocs.io/en/latest/system/arm/sabrelite.html)

After this, I will go through creating new QEMU devices and developing drivers and userspace applications for them, in [the following](https://straxy.github.io/2022/05/19/linux-device-driver-development-qemu/) post series.

<br />

Quick links to other posts in the series:

* [Part 1 - Basics](https://straxy.github.io/2021/10/09/qemu-board-emulation-part-1-basics/) : covers basic steps for preparing the development environment, downloading and building U-Boot bootloader and Linux kernel
* [Part 2 - Running the system](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/) : covers root filesystem handling and booting from SD card or over network (TFTP+NFS)
* [Part 3 - Vexpress GUI](https://straxy.github.io/2022/02/06/qemu-board-emulation-part-3-vexpress-gui/) : covers Linux kernel configuration in an attempt to display graphics
* [Part 4 - Vexpress Yocto](https://straxy.github.io/2022/04/23/qemu-board-emulation-part-4-vexpress-yocto/) : covers Yocto configuration for parts 2 and 3 of the blog series, so distribution and image can be built
