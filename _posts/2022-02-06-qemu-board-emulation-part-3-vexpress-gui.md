---
layout: post
title: Emulating Ubuntu GUI on Vexpress in QEMU
tags: ubuntu graphics linux qemu
---

*This is part 3 of the QEMU Board Emulation post series.*

In the [previous post](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/) the complete boot procedure from SD card or network boot has been presented.

In this post I will cover the following things
1. [Modifying Linux kernel configuration](#kernel)
2. [Enabling graphics in Ubuntu minimal](#ubuntu)

The goal is to use the emulated Versatile express board to display some simple graphics. This is also a very good feature of QEMU, where graphical applications can be run for some of the emulated boards.

# Modifying Linux kernel configuration {#kernel}

Versatile Express V2P-CA9 has the PL111 LCD display controller which is emulated in QEMU. The LCD controller is connected to a SiI9022 display bridge controlled over an I2C bus, and a Versatile display panel. However, none of it is not enabled by default in the Linux configuration.

In order to enable it, several configuration options in the linux kernel should be enabled

* `CONFIG_DRM_PL111` - enable LCD display controller
* `CONFIG_I2C_VERSATILE` - enable I2C bus support; the display bridge is on the I2C bus
* `CONFIG_DRM_SII902X` - enable display bridge
* `CONFIG_DRM_PANEL_ARM_VERSATILE` - enable ARM versatile display panel

Besides the hardware dependencies, in order to enable framebuffer and display image on the display, several additional options must be enabled

* `CONFIG_FB` - enable framebuffer support
* `CONFIG_FB_ARMCLCD` - enable ARM LCD framebuffer support
* `CONFIG_FRAMEBUFFER_CONSOLE` - enable console to be shown on the framebuffer
* `CONFIG_LOGO` - show tux logo when booting up
   
Enabling these configuration options is done using `menuconfig`. First the environment script `env.sh` should be sourced and then the menuconfig interface can be entered using

```bash
# Enter menuconfig
$ make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- O=build_vexpress menuconfig
```

This will show an interface like

![menuconfig interface](https://blogger.googleusercontent.com/img/a/AVvXsEgYr_1v4C3cP-BcR4MoE71r8QmKVMZWKt-5ydILJxxiF9PiX7OBG2CvLvuvRJm_EhhvUctUQnSjwKQXrDU_nI4VO8Qx_ss-pn2FzImV06jaYP8gJvEzDSt-IX2edWAdaOonwnW3kgjdoA8IhytCmbhXrVjbACH_2eTdqR_mNIy4cuWJrISgR0pIPRJX=s720)

The `menuconfig` can be traversed using arrow keys. Options are selected using the `space` key.

There are two types of options that can be selected

* boolean
  * `[ ]` - option is not enabled
  * `[*]` - option is enabled
* tristate
  * `< >` - option is not enabled
  * `<M>` - functionality will be compiled as a kernel module
  * `<*>` - functionality will be compiled in the linux kernel image

There are other filed types that are available, but they are not important for our analysis.

If we want to search for a keyword in menuconfig, we can press `/` and then type the keyword. This will open a window where selection can be made using a number on the keyboard.

<br />

Since we know which configuration options we want, we can search for them. Otherwise, they can be found in the following paths

```
CONFIG_DRM_PL111
-> Device Drivers
  -> Graphics support
    <*> DRM Support for PL111 CLCD Controller
```
```
CONFIG_I2C_VERSATILE
-> Device Drivers
  -> I2C support
    -> I2C Hardware Bus support
      <*> ARM Versatile/Realview I2C bus support
```
```
CONFIG_DRM_SII902X
-> Device Drivers
  -> Graphics support
    -> Display Interface Bridges
      <*> Silicon Image sii902x RGB/HDMI bridge
```
```
CONFIG_DRM_PANEL_ARM_VERSATILE
-> Device Drivers
  -> Graphics support
    -> Display Panels
      <*> ARM Versatile panel driver
```
```
CONFIG_FB
-> Device Drivers
  -> Graphics support
    -> Frame buffer Devices
      <*> Support for frame buffer devices  --->
```
```
CONFIG_FB_ARMCLCD
-> Device Drivers
  -> Graphics support
    -> Frame buffer Devices
      -> Support for frame buffer devices
        <*> ARM PrimeCell PL110 support
```
```
CONFIG_FRAMEBUFFER_CONSOLE
-> Device Drivers
  -> Graphics support
    -> Console display driver support
      [*] Framebuffer Console support
```
```
CONFIG_LOGO
-> Device Drivers
  -> Graphics support
    [*] Bootup logo
```

After the kernel configuration is modified, kernel has to be rebuilt again and all related files copied to the SD card (kernel image and kernel modules). For detailed instructions consult [part 1](https://straxy.github.io/2021/10/09/qemu-board-emulation-part-1-basics/) and [part 2](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/) of this series. If scripts from [github](https://github.com/straxy/qemu-board-emulation) are used, then `prepare-qemu.bash` script has to be executed after the Linux kernel is rebuilt in order to pull new image and kernel modules to the SD card.

## Updating kernel config manually

If scritps from [github](https://github.com/straxy/qemu-board-emulation) are used for testing, the supplied [defconfig](https://github.com/straxy/qemu-board-emulation/blob/main/defconfig) file can be used to update the kernel configuration to support display.

The `defconfig` needs to be copied into the `$PROJ_DIR/linux/build_vexpress/` directory and renamed to `.config`. After that the `olddefconfig` command has to be executed.

```bash
# Using defconfig
$ cp defconfig $PROJ_DIR/linux/build_vexpress/.config
$ make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- O=build_vexpress olddefconfig
```

After this the build can be started and `prepare-qemu.bash` script can be used to prepare the new SD card.

# Enabling graphics in Ubuntu minimal {#ubuntu}

In order to enable graphics in the Ubuntu minimal image, we need to boot the image and install apropriate packages.

QEMU can be started in the following manner in order to use image from SD card and allow network connection and use graphics for display

```bash
# Run QEMU with SD card and networking
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel $UBOOT \
                  -drive file=sd.img,format=raw,if=sd \
                  -net nic -net tap,ifname=qemu-tap0,script=no \
                  -serial mon:stdio
```

> _**NOTE:**_ It is assumed that network is enabled in the host according to [previous instructions](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/#network).

Please note that the command for starting QEMU has a significant difference compared to instructions from [part 1](https://straxy.github.io/2021/10/09/qemu-board-emulation-part-1-basics/) and [part 2](https://straxy.github.io/2022/01/25/qemu-board-emulation-part-2-running/#network) of blog series: instead of passing parameter `-nographics` the parameter `-serial mon:stdio` is used.

This will enable graphical window to appear and serial output for U-Boot will still go to terminal window from where QEMU is started. Serial output direction for Linux kernel can be controlled by specifying `console` parameter in the `bootargs` U-Boot variable:

* if `console=ttyAMA0` linux console output will go to the same terminal as U-boot output,
* if `console=tty0` linux console output will go to the graphical window.

First check if everything is working ok can be done by looking at the display while the system is booting, it should show a tux logo

![Tux logo](https://blogger.googleusercontent.com/img/a/AVvXsEj8pRwKmNF7GVBrdMoxYz3H_UKVLpxaRwe4ONyI-DWUC0JK07cQDvN7EuY-CbU89Z2a7RfMXjT0uQ42IxgvhH38S4B-zC4l43c-IxnySuHui1gYz2ULoGnVczDO_krQro7-x3ItkXXUBMwpDir18a8g0KAJNg5wd7KLpat9BY8-VYIRw6ia4XWk2ls-=s720)

An addition check if framebuffer support is working can be done by writing random data to framebuffer, which should show a 'noisy' image.

```bash
# Write random data to framebuffer
$ sudo cat /dev/urandom > /dev/fb0
```

Following image is the result

![Random noise](https://blogger.googleusercontent.com/img/a/AVvXsEhsLKQNaMm__MweUmRYWIuz6Uw26sTqUGmUm8F4A2NE5YfYFHHHjpmsuExlS3sAKo7d1_xaRNjI4HW9bnmBbxTe_AJkvX3mlo1HFVE2AXoL0_1s3m7hsTgFYa3CSIlzyR1DcXx78Mv_-vzPYJ9KpW3enM24n3YrneSMBVMWfrjRaUmU5F9Rsep96Ekq=s720)

<br />

After network is enabled in the emulated board then `apt-get` can be used to update and download necessary applications ([this post](https://askubuntu.com/a/1256290) from `askubuntu.com` has been used as a reference for installation of packages).

```bash
# Install xserver apps
$ sudo apt-get update
$ sudo apt-get install xserver-xorg-core --no-install-recommends --no-install-suggests
$ sudo apt-get install openbox --no-install-recommends --no-install-suggests
$ sudo apt-get install xinit
$ sudo apt-get install slim
```

In order to display graphics on the graphics window following command can be used

```bash
# Start graphics
$ sudo service slim start
```

This will show following login screen and, after logging in, an application running (`xterm`). Please note that this is bare minimum, so not much can be done without installing additional applications.

![Slim login](https://blogger.googleusercontent.com/img/a/AVvXsEjyySuicfrs0-RO1yWMMki0U7KZkGt-s5SoOHbDK6rCjHigk_oJJr6pKBL9gmFLD7mpodVnlBb2LfaQS6c91wPL1k1UygQEwp3FHGrSqyYfcz7qqktk44uaB9PardJKpSSvbWPDK-oqF1TWOjgf0v3GwzTBbTcSju5Sk8FkYexyqFqJ3vZGGVbJHeFV=s720)

![Login screen](https://blogger.googleusercontent.com/img/a/AVvXsEj_cOiLB8Ik5JrOXutMuQ3ohAHPfd1xnoG6KxzHYVzG9tOEGdzAlS3Yk2qXN0E1MiE6ApIsUrK3blPMWNqurBlyVT_dpoRlHFdir1YXMeA_udyKUCvGaDPeZm52IzYQMqy7isDgXjvW_2HZTp1fWeba8fbeT-bn0rUfE9OS9YfSsI4tJGhWSufWqEoZ=s720)

# Summary

In this post a way to configure Linux kernel to enable graphics support for Versatile Express board is shown. Enabling on the root filesystem side can be optimized by using a root filesystem image built by Yocto or Buildroot.
