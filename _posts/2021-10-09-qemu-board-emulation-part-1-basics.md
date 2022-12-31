---
layout: post
title: Preparing U-Boot and Linux kernel for QEMU Board emulation
tags: linux qemu u-boot
---

*This is part 1 of the QEMU Board Emulation post series.*

In the [previous post](https://straxy.github.io/2020/12/29/qemu-board-emulation-part-0/) I presented some of the goals for doing this work with QEMU.

In this post I will cover the following things
<!--more-->
- [Building QEMU](#building-qemu)
  - [Downloading source code](#downloading-source-code)
  - [Configuring and building](#configuring-and-building)
  - [Running QEMU](#running-qemu)
- [Getting toolchain](#getting-toolchain)
- [Building U-Boot](#building-u-boot)
  - [Downloading source code](#downloading-source-code-1)
  - [Configuring and building](#configuring-and-building-1)
  - [Running U-Boot inside QEMU](#running-u-boot-inside-qemu)
- [Building Linux kernel](#building-linux-kernel)
  - [Downloading source code](#downloading-source-code-2)
  - [Configuring and building](#configuring-and-building-2)
  - [Running U-Boot and Linux inside QEMU](#running-u-boot-and-linux-inside-qemu)
- [Simple "ramdisk"](#simple-ramdisk)
  - [Hello, world!](#hello-world)
- [Summary](#summary)

At the end of this post we will have a working QEMU, and starting point for U-Boot and Linux kernel which can be used for further work. A simple init ramdsik will be used instead of root filesystem, so the Linux kernel does not panic on boot. In the next post a real root filesystem will be used.

The Versatile Express V2P-CA9 will be used as the target board.

Before we start with the development, prerequisites need to be installed.
I am using Ubuntu 20.04 as the host system, so if different system is used it is possible that some additional packages from prerequisites need to be installed.

```bash
# Installing prerequisites
$ sudo apt -y install git libglib2.0-dev libfdt-dev libpixman-1-dev \
    zlib1g-dev libnfs-dev libiscsi-dev git-email libaio-dev \
    libbluetooth-dev libbrlapi-dev libbz2-dev libcap-dev \
    libcap-ng-dev libcurl4-gnutls-dev libgtk-3-dev libibverbs-dev \
    libjpeg8-dev libncurses5-dev libnuma-dev librbd-dev \
    librdmacm-dev libsasl2-dev libsdl2-dev libseccomp-dev \
    libsnappy-dev libssh2-1-dev libvde-dev libvdeplug-dev \
    libxen-dev liblzo2-dev valgrind xfslibs-dev kpartx libssl-dev \
    net-tools python3-sphinx libsdl2-image-dev flex bison \
    libgmp3-dev libmpc-dev device-tree-compiler u-boot-tools bc git \
    libncurses5-dev lzop make tftpd-hpa uml-utilities \
    nfs-kernel-server swig ninja-build libusb-1.0-0-dev
```

The development will be done in the `/home/user/qemu_devel` directory, and that directory will be refered to as `$PROJ_DIR`.

<br />

# Building QEMU

The latest stable version at the time of writing is 6.1.0.

## Downloading source code

The QEMU emulator source code can be obtained as an archive from [here](https://www.qemu.org/download/) or as a git repository from [github](https://github.com/qemu/qemu).

If archive is used, then QEMU can be downloaded and prepared using

```bash
# archive
$ wget -c https://download.qemu.org/qemu-6.1.0.tar.xz
$ tar xf qemu-6.1.0.tar.xz && mv qemu-6.1.0 qemu
$ cd qemu
```

If github is used, then QEMU can be downloaded and prepared using

```bash
# github
$ git clone https://github.com/qemu/qemu.git
$ cd qemu
$ git checkout v6.1.0 -b devel
$ git submodule init
$ git submodule update --recursive
```

## Configuring and building

Before building QEMU, it needs to be configured. In this process various options can be selected.
For these posts following configuration command will be used

```bash
# configuration
$ mkdir -p bin/arm && cd bin/arm
$ ../../configure --target-list=arm-softmmu \
                  --enable-sdl \
                  --enable-tools \
                  --enable-fdt \
                  --enable-libnfs
```

SDL is selected as GUI backend, QEMU tools for handling image and network will be compiled, device tree support and NFS support will also be included.

<br />

After the configuration step is done, code can be compiled using

```bash
# building
$ make -j4
```

The output files will be in `./arm-softmmu` directory, where the most import one is `qemu-system-arm`. The tools, like `qemu-img`, will be in the `./` directory.

In order to keep all of the details in one place, all relevant paths and exports will be saved to a file called `env.sh` in the root of the project.

So, after compiling the QEMU the `$PROJ_DIR/env.sh` should look like

```bash
# Environment file
PROJ_DIR=/home/user/qemu_devel

export PATH=$PROJ_DIR/qemu/bin/arm/arm-softmmu:$PROJ_DIR/qemu/bin/arm:$PATH
```

## Running QEMU

QEMU has many options which can be selected at runtime. Some of them are shown in the following table
<table style="border: 1px solid black; width: 100%;">
  <thead>
    <tr>
    <th>switch</th>
    <th>description</th>
    <th>example value</th>
  </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>-M</code></td>
      <td>select machine that will be emulated</td>
      <td><code>vexpress-a9</code>, <code>sabrelite</code></td>
  	</tr>
    <tr>
      <td><code>-m</code></td>
      <td>set amount of RAM memory</td>
      <td><code>512M</code>, <code>1G</code></td>
    </tr>
    <tr>
      <td><code>-kernel</code></td>
      <td>executable file that will be loaded</td>
      <td>u-boot or kernel ELF file</td>
    </tr>
    <tr>
      <td><code>-drive</code></td>
      <td>specify storage drive to be used</td>
      <td><code>file=sd.img,format=raw,if=sd</code></td>
    </tr>
    <tr>
      <td><code>-device</code></td>
      <td>specify device to be allocated</td>
      <td><code>loader,file=zImage,addr=0x62000000,force-raw</code></td>
    </tr>
    <tr>
      <td><code>-net</code></td>
      <td>ethernet network</td>
      <td><code>nic</code>, <code>tap,ifname=tap0,script=no</code></td>
    </tr>
    <tr>
      <td><code>-nographic</code></td>
      <td>disable display window</td>
      <td><code>N/A</code></td>
    </tr>
    <tr>
      <td><code>-serial</code></td>
      <td>set how serial interface is connected</td>
      <td><code>stdio</code>, <code>pty</code></td>
    </tr>
  </tbody>
</table>

The list of all switches can be obtained with 

```bash
qemu-system-arm --help
```

and available values for a specific switch using 

```bash
qemu-system-arm <switch> ?
```

Before using QEMU we need an executable file to run, so we will proceed to obtaining toolchain and building U-Boot.

# Getting toolchain

The cross-compilation toolchain for ARM architecture can obtained in various ways: get a prebuilt from [ARM/Linaro](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain) or [Bootlin](https://toolchains.bootlin.com/), or build a custom toolchain using [crosstool-NG](https://crosstool-ng.github.io/).

In this post we will be using a prebuilt toolchain from ARM. It can be downloaded using

```bash
# Download toolchain
$ wget -c https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf.tar.xz
$ tar xf gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf.tar.xz
```

Update the `$PROJ_DIR/env.sh` file so it looks like

```bash
# Environment file
PROJ_DIR=/home/user/qemu_devel

export PATH=$PROJ_DIR/gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf/bin:$PROJ_DIR/qemu/bin/arm/arm-softmmu:$PROJ_DIR/qemu/bin/arm:$PATH
```

This way, before cross-compiling any part of the code it is enough just to source the `$PROJ_DIR/env.sh` script.

# Building U-Boot

Even though QEMU can be used without the bootloader, where Linux kernel image, device tree blob and kernel command line are passed, in this blog series the goal is to emulate also the boot process. We will use [U-Boot](https://www.denx.de/wiki/U-Boot) as bootloader, but [Barebox](https://www.barebox.org/) can also be used.

## Downloading source code

As with QEMU, the source code can be obtained in the form of an [archive](https://ftp.denx.de/pub/u-boot/) or from a [github](https://github.com/u-boot/u-boot) repository.

> **_NOTE:_**  Version 2021.04 is the last one that supports this Vexpress board, so it is the version that is used.

If archive is used, then U-Boot source code can be downloaded and prepared using

```bash
# archive
$ wget -c https://ftp.denx.de/pub/u-boot/u-boot-2021.04.tar.bz2
$ tar xf u-boot-2021.04.tar.bz2 && mv u-boot-2021.04 u-boot
$ cd u-boot
```

If github is used, then U-Boot source code can be downloaded and prepared using

```bash
# github
$ git clone https://github.com/u-boot/u-boot.git
$ cd u-boot
$ git checkout v2021.04 -b devel
```

## Configuring and building

Before compiling U-Boot the environment script that was created needs to be sourced in order to add toolchain executables to the `$PATH`

```bash
# sourcing environment script
$ source $PROJ_DIR/env.sh
```

The configuration for Versatile Express V2P-CA9 board is done using the following command

```bash
# Configure U-Boot
$ make CROSS_COMPILE=arm-none-linux-gnueabihf- O=build_vexpress vexpress_ca9x4_defconfig
```

If additional adjustment needs to be made, it can be done using the `menuconfig` command as

```bash
# Configure U-Boot
$ make CROSS_COMPILE=arm-none-linux-gnueabihf- O=build_vexpress menuconfig
```

Once configuration is done, the build is started using

```bash
# Build U-Boot
$ make CROSS_COMPILE=arm-none-linux-gnueabihf- O=build_vexpress -j4
```

After the build is completed, in the `$PROJ_DIR/u-boot/build_vexpress` directory there will be a file called `u-boot` which will be run inside QEMU. For simpler handling, the path to this file can be added into the environment file, so next time it is sourced we will be able to access `u-boot` executable from anywhere. So after exporting this path the `$PROJ_DIR/env.sh` should look like

```bash
# Environment file
PROJ_DIR=/home/user/qemu_devel

export PATH=$PROJ_DIR/gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf/bin:$PROJ_DIR/qemu/bin/arm/arm-softmmu:$PROJ_DIR/qemu/bin/arm:$PATH

export UBOOT=$PROJ_DIR/u-boot/build_vexpress/u-boot
```

## Running U-Boot inside QEMU

In order to run U-Boot in QEMU the `u-boot` ELF file needs to be passed with the `-kernel` switch. Since at this moment only U-Boot is ready, we will be able to enter U-Boot and look around the provided console interface.

The command that can be used to run U-Boot inside QEMU is (do not forget to source the `$PROJ_DIR/env.sh` file beforehand)

```bash
# Run U-Boot in QEMU
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel $UBOOT -nographic
U-Boot 2021.04 (Aug 30 2021 - 01:45:32 +0200)

DRAM:  1 GiB
WARNING: Caches not enabled
Flash: 128 MiB
MMC:   MMC: 0
*** Warning - bad CRC, using default environment

In:    serial
Out:   serial
Err:   serial
Net:   smc911x-0
Hit any key to stop autoboot:  0 
```

After testing, exit QEMU with `Ctrl+A`,`x`.

# Building Linux kernel

Latest stable kernel at the time of writing is 5.14.3

## Downloading source code

The code can be obtained from [git server](git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git) or from an [archive](https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.14.3.tar.xz).

If archive is used, then Linux source code can be downloaded and prepared using

```bash
# archive
$ wget -c https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.14.3.tar.xz
$ tar xf linux-5.14.3.tar.xz && mv linux-5.14.3 linux
$ cd linux
```

If git server is used, then Linux source code can be downloaded and prepared using
```bash
# git server
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
$ cd linux
$ git checkout v5.14.3 -b devel
```

## Configuring and building

Before compiling Linux, the environment script that was created needs to be sourced in order to add toolchain executables to the `$PATH`

```bash
# sourcing environment script
$ source $PROJ_DIR/env.sh
```

The configuration for Versatile Express V2P-CA9 board is done using the following command

```bash
# Configure Linux kernel - multi_v7
$ make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- O=build_vexpress multi_v7_defconfig
```

The `multi_v7_defconfig` is a universal configuration for many ARMv7 based boards, where actual configuration is done based on the Device Tree.

If additional adjustment needs to be made, it can be done using the `menuconfig` command as

```bash
# Configure Linux - manual
$ make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- O=build_vexpress menuconfig
```

Once configuration is done, the build is started using

```bash
# Build Linux kernel
$ make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- O=build_vexpress -j4
```

After the build is completed, two files will be needed for running in the QEMU:

* file `zImage` in the `$PROJ_DIR/linux/build_vexpress/arch/arm/boot` directory - compressed Linux kernel image file
* file `vexpress-v2p-ca9.dtb` in the `$PROJ_DIR/linux/build_vexpress/arch/arm/boot/dts` directory - compiled device tree file with hardware description used by Linux kernel to set up hardware.
 
For simpler handling, the path to these file can be added into the environment file. After exporting these paths the `$PROJ_DIR/env.sh` should look like

```bash
# Environment file
PROJ_DIR=/home/user/qemu_devel

export PATH=$PROJ_DIR/gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf/bin:$PROJ_DIR/qemu/bin/arm/arm-softmmu:$PROJ_DIR/qemu/bin/arm:$PATH

export UBOOT=$PROJ_DIR/u-boot/build_vexpress/u-boot

export ZIMAGE=$PROJ_DIR/linux/build_vexpress/arch/arm/boot/zImage

export DTB=$PROJ_DIR/linux/build_vexpress/arch/arm/boot/dts/vexpress-v2p-ca9.dtb
```

## Running U-Boot and Linux inside QEMU

The idea is to use U-Boot to start the Linux kernel, the same way it would have been done on the real board. Since we will not handle flash, SD card interface or network intferace in this post, we will use the 'loader' feature of the QEMU to place the Linux kernel image and Device Tree file at the appropriate addresses in memory, as if the U-Boot code had already copied them from some of the possible bootable locations. In some of the other posts other methods will be covered.

The command that can be used to run U-Boot inside QEMU, with loading Linux kernel image and device tree blob at the appropriate addresses is (do not forget to source the `$PROJ_DIR/env.sh` file beforehand)

```bash
# Run U-Boot and Linux kernel in QEMU
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel $UBOOT -nographic \
                  -device loader,file=$ZIMAGE,addr=0x62000000,force-raw=on \
                  -device loader,file=$DTB,addr=0x68000000,force-raw=on
U-Boot 2021.04 (Aug 30 2021 - 01:45:32 +0200)

DRAM:  1 GiB
WARNING: Caches not enabled
Flash: 128 MiB
MMC:   MMC: 0
*** Warning - bad CRC, using default environment

In:    serial
Out:   serial
Err:   serial
Net:   smc911x-0
Hit any key to stop autoboot:  0 
```

U-Boot needs to be stopped and then following commands need to be entered so Linux kernel is started with the device tree.

```bash
# Run Linux kernel from U-Boot
U-Boot&gt; setenv bootargs "console=ttyAMA0"
U-Boot&gt; bootz 62000000 - 68000000
Kernel image @ 0x62000000 [ 0x000000 - 0x93f200 ]
## Flattened Device Tree blob at 68000000
   Booting using the fdt blob at 0x68000000
   Loading Device Tree to 7fe6d000, end 7fe7375c ... OK

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.14.3 ...
...
[    3.208263] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

The first command sets boot arguments that are passed to the Linux kernel. In this case, the only thing that needs to be configured is the device that is used for serial console, and that is the `ttyAMA0`, or the UART0 port.

The second command start the boot of the Linux kernel, where parameters that are passed are address of `zImage` kernel image, address of init ramdisk (in this case `-` since it is not used) and the address where device tree blob is placed.

After executing these commands, the Linux kernel will panic since there is no root filesystem, which is expected.

# Simple "ramdisk"

What is actually a root filesystem? It is a set of files and directories organized in a certain way. Those files and directories can be on a physical medium (HDD, SD card, eMMC, Flash memory), a remote location (NFS boot), but also can be executed from RAM in case the init ramdisk/ramfs is used.

In all cases, the kernel is looking for an 'init' file which is the first one that is executed. So, by supplying an 'init' file we can give the kernel a reason not to panic.

The simplest way to create this 'init' file, without building a full-blown root filesystem, is to create a 'Hello, world!' application and link it statically. The application should write directly to registers (we will use UART peripheral so we can get some messages) and will be used only for the demonstration.

## Hello, world!

A <abbr title="Not exactly classic, we want the 'while(1)' at the end so the code loops forever">classic</abbr> 'Hello, world!' C application can be used:

```C++
/* hello.c */
#include <stdio.h>

void main()
{
    printf("Hello, world!\n");
    while(1);
}
```

The code can be compiled into a static binary using:

```bash
# Compile 'Hello, world!'
$ arm-none-linux-gnueabihf-gcc -static hello.c -o hello
```

The init ramdisk that can be used with U-Boot can be created using:

```bash
# Create ramdisk
$ echo hello | cpio -o -H newc > initrd
$ gzip initrd
$ mkimage -A arm -O linux -T ramdisk -d initrd.gz uRamdisk
```

The command that can be used to run U-Boot inside QEMU, with loading Linux kernel image, device tree blob and uRamdisk at the appropriate addresses is (do not forget to source the `$PROJ_DIR/env.sh` file beforehand)

```bash
# Run U-Boot and Linux kernel in QEMU with uRamdisk
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel $UBOOT -nographic \
                  -device loader,file=$ZIMAGE,addr=0x62000000,force-raw=on \
                  -device loader,file=$DTB,addr=0x68000000,force-raw=on \
                  -device loader,file=uRamdisk,addr=0x68080000,force-raw=on
U-Boot 2021.04 (Aug 30 2021 - 01:45:32 +0200)

DRAM:  1 GiB
WARNING: Caches not enabled
Flash: 128 MiB
MMC:   MMC: 0
*** Warning - bad CRC, using default environment

In:    serial
Out:   serial
Err:   serial
Net:   smc911x-0
Hit any key to stop autoboot:  0 
```

U-Boot needs to be stopped and then following commands need to be entered so Linux kernel is started with the device tree and uRamdisk.

```bash
# Run Linux kernel from U-Boot with ramdisk
U-Boot> setenv bootargs "root=/dev/ram rdinit=/hello console=ttyAMA0"
U-Boot> bootz 62000000 68080000 68000000
Kernel image @ 0x62000000 [ 0x000000 - 0x93f200 ]
## Loading init Ramdisk from Legacy Image at 68080000 ...
   Image Name:   
   Image Type:   ARM Linux RAMDisk Image (gzip compressed)
   Data Size:    1183353 Bytes = 1.1 MiB
   Load Address: 00000000
   Entry Point:  00000000
   Verifying Checksum ... OK
## Flattened Device Tree blob at 68000000
   Booting using the fdt blob at 0x68000000
   Loading Ramdisk to 7fd52000, end 7fe72e79 ... OK
   Loading Device Tree to 7fd4b000, end 7fd5175c ... OK

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.14.3 ...
...
[    2.840502] Run /hello as init process
Hello, world!
```

In kernel arguments there are now two additional

* `root=/dev/ram`, indicating that root filesystem will be in RAM memory, where init ramdisk is extracted,
* `rdinit=/hello`, overriding the default `init` program that is used from ramdisk so our `hello` is used.

Once kernel boots, it will run the `hello` application and print "Hello, world!".

# Summary

In this post the basic steps building U-Boot and Linux kernel were covered. This is still far from the actual use-case for an embedded Linux system, since root filesystem is missing.

The root filesystem will be covered in the [next post](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running"), together with steps for running bootloader and Linux kernel from different mediums, which will be similar to the way and embedded Linux is used on real development boards.
