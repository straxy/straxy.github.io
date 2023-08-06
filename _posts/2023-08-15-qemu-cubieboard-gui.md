---
layout: post
title: Interactive QEMU Cubieboard
tags: qemu cubieboard gui ps2
---

*This is part 6 of the QEMU Board Emulation post series.*

Posts in this post series so far have been focused on the Versatile Express board, since it has wide range of supported peripherals and is easy to start with.

I have recently worked on contributing to Allwinner A10 and Cubieboard support in QEMU. I have added several components (clock controller module and DRAM controller) as well as I2C controller in order to be able to execute U-Boot and SPL code from SD card. By reusing the bootrom approach from Allwinner H3, the Cubieboard can now be started just by supplying the SD card image, and it will perform boot similar to the one executed on a real board.

In this blog post I will focus on Yocto support for Cubieboard and how to use new QEMU support. I will also extend Cubieboard with the previously developed I2C sensor peripheral, to test that application can be shared between Vexpress and Cubieboard.

<!--more-->
<br />

Items that will be covered in this post are

- [Yocto for Cubieboard](#yocto-for-cubieboard)
  - [Organizing layers](#organizing-layers)
  - [Building](#building)
- [Testing SD card boot](#testing-sd-card-boot)
- [Testing I2C sensor](#testing-i2c-sensor)
- [Summary](#summary)

Sources for the Yocto build environment for Cubieboard can be found in following repositories:

* [https://github.com/straxy/qemu-sunxi-yocto](https://github.com/straxy/qemu-sunxi-yocto) - manifest repository used to initialize Yocto environment
* [https://github.com/linux-sunxi/meta-sunxi](https://github.com/linux-sunxi/meta-sunxi) - official sunxi OpenEmbedded layer for Allwinner-based boards
* [https://github.com/straxy/meta-mistra-base](https://github.com/straxy/meta-mistra-apps) - `meta-mistra-base` layer, containing distribution definition - `kirkstone` branch
* [https://github.com/straxy/meta-mistra-apps](https://github.com/straxy/meta-mistra-apps) - `meta-mistra-apps` layer, containing image information and I2C sensor application

> Note: The organization has been updated for Vexpress as well in kirkstone brances, so the only difference are the manifest repositories (qemu-vexpress-yocto vs qemu-sunxi-yocto) and platform support layers (meta-vexpress vs meta-sunxi).

# Yocto for Cubieboard

The official `meta-sunxi` layer (available at [https://github.com/linux-sunxi/meta-sunxi](https://github.com/linux-sunxi/meta-sunxi)) supports Cubieboard out-of-the-box, so the `machine` part is already prepared (unlike the Versatile Express board, where we had to create our `meta-vexpress` to contain the `machine` information).

## Organizing layers

We can reuse the `meta-mistra-base` and `meta-mistra-apps` layers that were used for Versatile Express board demonstration (in [part 4](https://straxy.github.io/2022/04/23/qemu-board-emulation-part-4-vexpress-yocto/) and [part 5](https://straxy.github.io/2022/11/05/qemu-board-emulation-part-5-vexpress-yocto-qt6/) of post series).

> Note: the layer organization in parts 4 and 5 was a bit different - `meta-vexpress` had both the `machine` and `distro` specification, while `meta-vexpress-apps` had applications; the design in `kirkstone` branch for Versatile Express has been updated to use the `meta-mistra-base` and `meta-mistra-apps` layers.

The `meta-mistra-base` layer contains the distribution definition (same distributions, `mistra-framebuffer` and `mistra-x11`), while the `meta-mistra-apps` contains `i2c-sens` application and `mistra-image` definition.

## Building

Building is similar to the way it was done for Versatile Express board.

After initializing the `qemu-sunxi-yocto` repo

```bash
$ repo init -u https://github.com/straxy/qemu-sunxi-yocto -m default.xml
$ repo sync
```

the environment is initialized using

```bash
$ source setup-environment build_fb
```

The build for framebuffer distro is done using

```bash
DISTRO=mistra-framebuffer MACHINE=cubieboard bitbake mistra-image
```

The final result is in `./tmp/deploy/images/cubieboard`, the file with extension `*.sunxi-sdimage`. This is the SD card image which can be either copied (`dd`) to an SD card image created with `qemu-img`, or can be just resized using `qemu-img` to a power of 2 size. For instance

```base
$ qemu-img resize ./tmp/deploy/images/cubieboard/mistra-image.sunxi-sdimage 1G
```

# Testing SD card boot

The patches that allow Cubieboard SD card boot are in latest QEMU ([patch series](https://patchew.org/QEMU/20221226220303.14420-1-strahinja.p.jankovic@gmail.com/) and [specific code change](https://github.com/qemu/qemu/commit/bb9271cadb0d0b32d566619606fedc9c08f612cc)).

In order to do the boot, the QEMU should be called without the `-kernel` parameter, and only with `-sd` parameter.

```bash
$ qemu-system-arm -M cubieboard -nographic -sd mistra-image.sunxi-sdimage
```

After this, the emulated board will go through the regular boot process: SPL -> U-Boot -> Linux, and we will be presented with the prompt to login to the image.

![boot sequence](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjJ54ObPEth6DoGTwftzj6s2Vc9gberyjL3lwwolc5rzkul2ylfQIXhDIdVQb5B710hcrEXZGZ1zP9BaV-ehKuG5-Iht5zOMaGT355p4wMKrz4lxYzieYykkMjQ3ad2htg7tSwd8Nf5Yqsk0UhMo4pOFl2g_CPtukugSB39r_-ozrP24q7GUOmbbdRl/s16000/WindowsTerminal_YJo9Lok0x2.gif)


# Testing I2C sensor

In order to attach the I2C sensor to Cubieboard, few lines (similar to ones used for Vexpress) have to be added to `cubieboard.c`.

```c
  i2c = I2C_BUS(qdev_get_child_bus(DEVICE(&a10->i2c0), "i2c"));
  i2c_slave_create_simple(i2c, "mistra.i2csens", 0x36);
```

After this, when QEMU is started, we can test if I2C sensor is working using `i2c-tools`

```bash
# read ID register
$ i2cget -y 0 0x36 0x0
0x5a
# enable I2C component
$ i2cset -y 0 0x36 0x1 0x1
$ i2cget -y 0 0x36 0x1
0x01
# read data register
$ i2cget -y 0 0x36 0x2
0x2c
```

We can also use the old `i2csens-app` application from `meta-mistra-apps` to obtain similar results.

```bash
$ i2csens-app /dev/i2c-0
Measured 18
Measured 22.5
```

# Summary

In this post instructions for using Yocto with Cubieboard were shown.
Since QEMU Cubieboard supports SD card boot, that has been used for testing.
This can be very useful for initial testing, without available hardware.
