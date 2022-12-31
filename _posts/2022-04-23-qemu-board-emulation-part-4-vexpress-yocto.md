---
layout: post
title: Yocto for Vexpress A9 in QEMU
tags: yocto linux qemu
---

*This is part 4 of the QEMU Board Emulation post series.*

In parts [2](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/) and [3](https://straxy.github.io/2022/02/06/qemu-board-emulation-part-3-vexpress-gui/) of this post series the complete boot procedure from SD card has been presented, as well as how to configure kernel support required to enable graphical display for Vexpress-A9 board.

In both posts Ubuntu was used as root filesystem. Using Ubuntu as root filesystem is simple and fast to use, but it also has a lot of packages which are not necessary.

In this post I will cover the Yocto setup for Vexpress-A9 board. Using Yocto we will be able to build a custom distribution which will allow us to run Linux with or without GUI on QEMU Vexpress-A9.
<!--more-->
Items that will be covered are

- [Yocto introduction](#yocto-introduction)
- [Base Yocto setup](#base-yocto-setup)
- [Distribution configuration](#distribution-configuration)
- [Machine configuration](#machine-configuration)
- [U-boot recipe](#u-boot-recipe)
- [Linux kernel recipe](#linux-kernel-recipe)
- [Image recipe](#image-recipe)
- [Building and running images in QEMU](#building-and-running-images-in-qemu)
  - [Prerequisites](#prerequisites)
  - [Preparing environment](#preparing-environment)
  - [Building and running images](#building-and-running-images)
  - [Framebuffer image](#framebuffer-image)
  - [X11 image and Sato](#x11-image-and-sato)
- [Summary](#summary)

Sources for the Yocto build environment for Vexpress-A9 can be found in following repositories:

* [https://github.com/straxy/qemu-vexpress-yocto](https://github.com/straxy/qemu-vexpress-yocto) - base repository used to initialize Yocto environment
* [https://github.com/straxy/meta-vexpress](https://github.com/straxy/meta-vexpress) - `meta-vexpress` layer containing distro, machine and images configuration, as well as recipes for bootloader and linux kernel

# Yocto introduction

[Yocto](https://www.yoctoproject.org/) is tool used to build Board Support Package (BSP) and Linux distributions, especially for embedded targets. It is very configurable and provides fine grained control of output images.

The project is organized into layers and applications that can be built are described in recipes. Configuration for a build depends on selected image, machine and distribution, as well as local configuration parameters.

For more details about Yocto [Bootling Yocto training slides](https://bootlin.com/training/yocto/) can be used.

I tried to use the Freescale/NXP Yocto organization as a reference when creating these layers, so method of use is very similar to the method when working with iMX SoCs.

# Base Yocto setup

As stated previously, the base repository holds the manifest file and base configuration.

The manifest file describes all the layers that are used in the project. In this case only 'dunfell' branch is selected, but selection can be made on a specific commit.

```
<?xml version="1.0" encoding="UTF-8" ?>
  <manifest>
  <default revision="dunfell" sync-j="4"/>

  <remote fetch="git://git.yoctoproject.org" name="yocto"/>
  <remote fetch="https://github.com/straxy" name="vexpress"/>
  <remote fetch="https://github.com/openembedded" name="oe"/>

  <project name="poky" path="sources/poky" remote="yocto" revision="dunfell"/>
  <project name="meta-openembedded" path="sources/meta-openembedded" remote="oe" revision="dunfell"/>
  <project name="qemu-vexpress-yocto" path="sources/base" remote="vexpress" revision="main">
    <copyfile dest="setup-environment" src="scripts/setup-environment"/>
  </project>
  <project name="meta-vexpress" path="sources/meta-vexpress" remote="vexpress" revision="main"/>

  </manifest>
```

The `poky` and `meta-oe` layers provide base applications and images, which can be extended by the higher-level layers, like `meta-vexpress`.

The `setup-environment` script is used to initialize a build environment. Part of it initializes the `bblayers.conf` and `local.conf` based on the input files in the `templates` directory.

# Distribution configuration

Distribution configuration is in the `meta-vexpress` layer, in the [conf/distro](https://github.com/straxy/meta-vexpress/tree/main/conf/distro) directory.

Two distributions are configured:

- The framebuffer distribution which disables all graphical backends, like X11, Wayland, Vulkan. This way the output image size is reduced.
- The X11 distribution which uses the X11 server.

Selection of distribution is made at compile time by passing either `mistra-framebuffer` or `mistra-x11` to the `DISTRO` variable.

# Machine configuration

Machine configuration is in the `meta-vexpress` layer, in the [conf/machine](https://github.com/straxy/meta-vexpress/tree/main/conf/machine) directory.

Machine has definitions output images that should be built, as well as parameters for U-Boot and Linux kernel. This way, differences between machines can be kept in separate files and same distribution can be used for different machines.

# U-boot recipe

The goal with this exercise is to use the same software versions as in [part 2](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/) of the blog post series.

The `u-boot` related recipes are in [recipes-bsp](https://github.com/straxy/meta-vexpress/tree/main/recipes-bsp) directory.

The `u-boot` directory holds the main `u-boot` recipe, which targets a specific git commit in order to use the same version as in [part 2](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/) of the blog post series.

In the `u-boot-scr` directory is recipe used to build a u-boot script. The u-boot script is a special script run by U-Boot at boot. This way, all of the commands that were entered manually in [part 2](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/) of the blog post series will be automatically executed.

The commands are in [boot.cmd](https://github.com/straxy/meta-vexpress/blob/main/recipes-bsp/u-boot-scr/files/boot.cmd) file, and instructions on how this script can be manually built are in the `do_compile` step of the [u-boot-scr.bb](https://github.com/straxy/meta-vexpress/blob/main/recipes-bsp/u-boot-scr/u-boot-scr.bb) recipe.

# Linux kernel recipe

Linux recipe is stored in the [recipes-kernel](https://github.com/straxy/meta-vexpress/tree/main/recipes-kernel/linux). It selects the appropriate git commit in order to have the same version as in [part 2](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/) of the blog post series.

# Image recipe

Image recipe is stored in the [recipes-extended/images](https://github.com/straxy/meta-vexpress/blob/main/recipes-extended/images/core-image-test.bb) directory.

This is a copy of the `core-image-minimal` image supplied from Yocto, but can be extended if needed. This is an image that does not use graphics.

Another image that will be used is `core-image-sato`, which uses graphical environment.

# Building and running images in QEMU

## Prerequisites

Before Yocto image can be built, several packages must be installed:

```bash
$ sudo apt-get install gawk wget git-core diffstat unzip texinfo \
     gcc-multilib build-essential chrpath socat cpio python3 \
     python3-pip python3-pexpect xz-utils debianutils iputils-ping \
     python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3 \
     xterm
```

> _**NOTE**_: For details look at [Yocto reference manual](https://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#ubuntu-packages)

## Preparing environment

In order to prepare for a build, the manifest [repository](https://github.com/straxy/qemu-vexpress-yocto") must be used with `repo` tool.

```bash
$ repo init -u https://github.com/straxy/qemu-vexpress-yocto -m default.xml
$ repo sync
```

After the repo is initialized and synced, all recipe sources will be in the `sources` directory. Two additional directories will be created, `downloads`, where archives and git repositores with sources for packages are downloaded and kept, and `sstate-cache` intermediate build products for reuse will be stored during the build process. The `sstate-cache` directory can speed up rebuilds, or different flavor builds, since packages that are not changed will be reused.

The next step is to initialize the build environment

```bash
$ source setup-environment <build_dir>
```

where `<build_dir>` is custom directory where build will be performed and output files stored. After this command is executed current directory is automatically changed to `build_dir`.

## Building and running images

The build is started using the following command

```bash
# build command
$ DISTRO=<selected_distribution> MACHINE=<selected_machine> bitbake <selected_image>
```

Once build is completed, output image will be placed in `<build_dir>/tmp/deploy/images/<selected_machine>/<selected_image>-<selected_machine>.wic`. There will be also other build products, like u-boot binary (`u-boot.elf` will be used for running QEMU), linux kernel binary, etc.

Image is in `wic` format and can be copied to the SD card image using following commands

```bash
# create SD card image
$ qemu-img create sd.img 4G
# 'insert' SD card
$ sudo kpartx -av ./sd.img
# note the loopXX that is used and use dd to copy wic image
# if there is no output, look at $ losetup -a to find the loop device
$ sudo dd if=<selected_image>-<selected_machine>.wic of=/dev/loopXX bs=1M iflag=fullblock oflag=direct conv=fsync
# after copying is done 'remove' the SD card
$ sudo kpartx -d ./sd.img
```

Once SD card is ready, QEMU can be started using the following command

```bash
# Run QEMU with SD card and networking
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel u-boot.elf \
                  -drive file=sd.img,format=raw,if=sd \
                  -net nic -net tap,ifname=qemu-tap0,script=no \
                  -serial mon:stdio
```

## Framebuffer image

Distro that is used is `mistra-framebuffer` and image is `core-image-test`, so complete build command is

```bash
# framebuffer image build command
$ DISTRO=mistra-framebuffer MACHINE=vexpress-qemu bitbake core-image-test
```

After image is built, copied to the SD card and QEMU is run, the following output will be visible

```bash
# QEMU output
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel u-boot.elf \
                  -drive file=sd.img,format=raw,if=sd \
                  -net nic -net tap,ifname=qemu-tap0,script=no \
                  -serial mon:stdio
[ ... ]
Mistra FrameBuffer 3.1 vexpress-qemu /dev/ttyAMA0
vexpress-qemu login:
```

## X11 image and Sato

Distro that is used is `mistra-x11` and image is `core-image-sato`, so complete build command is

```bash
# framebuffer image build command
$ DISTRO=mistra-x11 MACHINE=vexpress-qemu bitbake core-image-sato
```

After image is built, copied to the SD card and QEMU is run, the following output will be visible

```bash
# QEMU output
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel u-boot.elf \
                  -drive file=sd.img,format=raw,if=sd \
                  -net nic -net tap,ifname=qemu-tap0,script=no \
                  -serial mon:stdio
[ ... ]
Mistra X11 3.1 vexpress-qemu /dev/ttyAMA0
vexpress-qemu login:
```

During the boot process following image is visible

![Sato image](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgmP3udWsXoExTDER62xCsR4QWqgMnBKNNtOSvvNwy8h2RabesALjyUfdT3ceqINoqvfl55E9jrO9r_QvBenrniF9eOg0-_yvvw_dkBLyxCCEREjFmxlNj0_5mZyUrTKGErn4FBkmMxeC5ra0pAeRlcQhpPih-PVprE5FGxfrHS7P24AMbUcxphq92z/s720/openembedded.png)

After the boot process is complete the environment is ready

![Desktop](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjpxeUZJsh2D5Iud-3QCepyzS9D3K5-GWytNBfEVrSdg6JjnIMKQlUmrHcDhFCdeGvFDofLXyE3_4VXc2wTlugVj8vFGhdxqxDHfzfAH0iPHM3w4_seWa4-9ofSn2pniL_4Tac1mV06PEknanu-tWKz5u-_WdXzmaIub3tkkv46J4NuUUj2GVNZv82k/s720/sato.png)

# Summary

In this blog post the procedure for using Yocto to build the distribution is shown. It can be further extended with other layers (like `meta-qt5`).
