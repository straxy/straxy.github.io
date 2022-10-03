---
layout: post
title: Learning Linux device driver development using QEMU - Introduction
tags: memory-mapped linux qemu i2c device-driver userspace
---

In the previous blog series [QEMU board emulation](https://straxy.github.io/2020/12/29/qemu-board-emulation-part-0/) I introduced simple methods to prepare and run a QEMU image for board emulation. Target board was Versatile express and mainline U-Boot and Linux kernel were used for testing. The goal of that blog series is to show how to use QEMU for board emulation.

<br/>

In this blog post series I will go deeper into the topics of device driver and userspace application development for embedded Linux systems.

We will continue using QEMU for testing. There are multiple benefits from using QEMU over some standard COTS board:

* it is cheaper, there is no need to buy additional hardware, also making it easier to try and follow
* 'new' hardware can be designed in QEMU, compared to COTS board where hardware cannot change and drivers are already available
  * even if we consider connecting peripherals over external parallel or serial busses as 'changing the COTS board', drivers already exist even for those peripherals
* it is easier to debug during development in QEMU

There is also a possibility to use FPGA-SoC boards, where custom peripherals can be created in the FPGA part, but it is much more complex and also requires the board to be available.

<br/>

The goals of this post series would be to show development of

* character device driver for a custom memory-mapped hardware emulated in QEMU
* userspace application that interacts with character device driver for memory-mapped peripheral
* userspace I2C application/driver used to interact with custom I2C peripheral

In both cases I will also cover details on how to write a simple memory-mapped and I2C peripherals in QEMU.

<br />

Quick links to other posts in the series:

* [Part 1 - Developing custom memory-mapped peripheral in QEMU](https://straxy.github.io/2022/06/25/linux-driver-qemu-part-1-custom-device/) : covers basic steps creating a new memory-mapped peripheral in QEMU, how to integrate it and simple testing.
* [Part 2 - Developing Linux device driver for QEMU custom memory-mapped peripheral](https://straxy.github.io/2022/08/14/linux-driver-qemu-part-2-char-driver/) : covers structure of a Linux device driver that can be used to initialize and interact with developed custom memory-mapped device.
