---
layout: post
title: Developing Linux device driver for QEMU custom memory-mapped device
tags: memory-mapped ubuntu yocto device-tree linux qemu device-driver
---

*This is part 2 of the Linux device driver development post series.*

In the [previous post](https://straxy.github.io/2022/06/25/linux-driver-qemu-part-1-custom-device/) I presented the steps for creating a custom memory-mapped peripheral in QEMU.

In this post I will cover the following things

1. [Developing character device driver for the designed memory-mapped device](#driver)
   * [Platform driver](#plat)
   * [Chardev operations](#rw)
   * [Sysfs attributes](#sysfs)
2. [Building driver out-of-tree](#module)
3. [Adding driver to Yocto](#yocto)
4. [Testing driver](#test)
5. [Summary](#summary)

# Developing character device driver for the designed memory-mapped device {#driver}

The driver for the custom memory-mapped device should provide interface for user space applications to use the custom-memory mapped peripheral. This means that all bit-fields can be read and modified by a user-space application.

The device first needs to be 'recognized' by the system for which we will use Device-tree mapping and platform driver structures.

After device is recognized by the system, we need to have some methods to access and modify registers of our custom memory-mapped peripheral, and we will use character device driver structures with sysfs attributes.

## Platform driver {#plat}

Peripherals can be connected to the processor directly (via 'platform' bus, like our memory-mapped peripheral) and also via external busses: I2C, SPI, UART, USB, PCI, etc. Some of these busses support dynamic enumeration of devices (USB, PCI), but for others there needs to be a way to let the system know what is present in those busses.

In standard PCs BIOS is in charge of preparing and providing information about present devices to the operating system on boot.

ARM systems do not have BIOS, so the information must be provided in some other way. Initially, it was done by hardcoding details for each board in the Linux architecture specific code, so when board runs it has all of the information about peripherals that it needs.

However, that also meant that two boards with just minor differences could not use the same Linux kernel image.

To avoid that, Device Tree specification is now used. In Device Tree specifics of a device are described: memory regions, interrupt line numbers, dma channels, as well as key for binding the compatible driver with that device.

This way the device driver gets all relevant information about the way the device is integrated into the system from the Device Tree.

### Device Tree description

Device Tree is a tree-like hierarchical description of system devices and busses they are connected to. It is used for describing peripherals that cannot be automatically enumerated.

Device Tree is written in textual form (`.dts` and `.dtsi` files, but also C header files can be preprocessed) and need to be translated into binary form (binary blob, `.dtb`) before they can be used on a board. At boot time Linux kernel (also recent versions of U-Boot) parse the Device Tree blob and try to match corresponding device drivers in order to initialize the system.

Without going into too much details (there are always good materials on [Bootlin website](https://bootlin.com/blog/device-tree-101-webinar-slides-and-videos/)), the Device Tree excerpt for our memory-mapped peripheral should look like

```
iofpga@7,00000000 {
...
    mmsens@18000 {
        compatible = "mistra,mmsens";
        reg = <0x18000 0x1000>;
        interrupts = <29>;
    };
...
};
```

The `compatible` string is used by the device driver when probing to match with this device.

Field `reg` is used to describe memory regions used by the device, in our case it is offset `0x18000` of the `CS7` region. Memory range is `0x1000` long.

The interrupt line that is used is noted in the `interrupts` field.

### Platform driver for memory-mapped peripheral

The platform driver provides callbacks that are called when Device Tree is parsed and device is probed, which are then used to get information about the device. The `platform_driver` structure has following important fields

* `.driver` - initialized with driver name and table of compatible strings used for matching driver with the device from the Device Tree
* `.probe` callback - called when Device Tree is parsed, to try to register device with the driver
* `.remove` callback - called when device is removed (not particularly interesting for platform drivers) or when driver is removed from the system

In the case of our memory-mapped device, `platform_driver` structure should look like

```c
static struct platform_driver mmsens_driver = {
	.driver = {
		.name = DRIVER_NAME,
		.of_match_table = mmsens_of_match,
	},
	.probe = mmsens_probe,
	.remove = mmsens_remove,
};
```

> _**NOTE**_: The `of_` prefix comes from 'Open Firmware', since full name of Device Tree is Open Firmware Device Tree.

Match table should contain the compatible string, which (from Device Tree excerpt above) is selected to be `"mistra,mmsens"`

```c
static const struct of_device_id mmsens_of_match[] = {
	{ .compatible = "mistra,mmsens", },
	{ /* end of table */ }
};
MODULE_DEVICE_TABLE(of, mmsens_of_match);
```

The `.probe` callback should do the following things:

1. Try to match device from Device Tree with the driver based on the 'compatible' string (`of_match_node`)
2. Try to extract memory regions information from Device Tree (`platform_get_resource`) and remap memory so it is accessible by the driver (`devm_ioremap_resource`)
3. Try to extract interrupt lines information from Device Tree (`platform_get_irq`) and register handler function for that interrupt (`devm_request_irq`)

Interrupt handler in our case should only clear the `IFG` flag

```c
static irqreturn_t mmsens_isr(int irq, void *data)
{
	struct mmsens *dev = data;

	pr_info("Interrupt received\n");

	iowrite32(0, dev->base_addr + MMSENS_STATUS_OFFSET);

	return IRQ_HANDLED;
}
```

In the simplest scenario the `.remove` callback does not need to do anything, but once we add the character device operations it will change.

## Chardev operations {#rw}

So far, the platform driver only allows us to match the driver with the device when Device Tree description is parsed.

In order to be able to interact with the device and read/write some data to it, we need add another layer, which is character device. The use of character device allows us later on to add more operations, like IOCTL or sysfs attributes, to have even more ways to interact with the device.

Character device operations are executed when character device file (usually under `/dev`) is accessed, so we need to make sure that information obtained from platform driver framework (base address of the remapped region) can be used within the character device operations. For that purpose, we will create a custom structure which will be stored as `private_data` and shared between these two frameworks

```c
/**
 * struct mmsens - mmsens device private data structure
 * @base_addr:	base address of the device
 * @irq:	interrupt for the device
 * @dev:	struct device pointer
 * @parent:	parent pointer
 * @cdev:	struct cdev
 * @devt:	dev_t member
 */
struct mmsens {
	void __iomem *base_addr;
	int irq;
	struct device *dev;
	struct device *parent;
	struct cdev cdev;
	dev_t devt;
};
```

The character device operations structure defines several callbacks

* `.open` - used to prepare driver structures for accessing the device
* `.release` - cleanup of operations done in `.open`
* `.read` - read operation from the device, usually raw data that is copied to the user space
* `.write` - write operation to the device, usually raw data that is copied from the user space

In the case of our memory-mapped device, it should look like

```c
static struct file_operations mmsensdev_fops = {
	.owner = THIS_MODULE,
	.open = mmsens_open,
	.release = mmsens_release,
	.read = mmsens_read,
	.write = mmsens_write,
};
```

The details of the implementation are available in the [github](https://github.com/straxy/mmsens-drv) repository.

## Sysfs attributes {#sysfs}

The character device operations allow only reading or writing of raw data to the device. However, since our device has several registers and supports different operations, we need an additional interface to be able to control it.

That can be achieved using IOCTL callback, or by using sysfs attributes (we will use the latter).

The sysfs attributes are created in the `/sys` directory when the device is created as separate files. Each file can be used according to the way they are specified (read-only, write-only, read-write) and they can be used to access individual bits/registers, or perform specific operations.

In order to be able to use the sysfs attributes, the character device class must be created, and all devices of that class will appear under that directory.

The attributes can be read and/or written. The `<operation>_show` callback is used when attribute file is read, `<operation>_store` callback is used when attribute file is written, and attribute is initialized using `static DEVICE_ATTR_RW(<operation>);` (there are also the `_RO` and `_WO` variants).

In the case of our device, following attributes are supported

* `DEVICE_ATTR_RO(interrupt)` - used to obtain interrupt status, and can be used to poll from user space application (more on that in next post)
* `DEVICE_ATTR_RW(enable_interrupt)` - used to enable/disable interrupt generation
* `DEVICE_ATTR_RW(enable)` - used to enable/disable device
* `DEVICE_ATTR_RW(frequency)` - used to select desired sampling frequency
* `DEVICE_ATTR_RO(available_frequencies)` - used to show available sampling frequencies
* `DEVICE_ATTR_RO(data)` - used to show data in BCD format

If we take a look at the `data` attribute for instance, we can see that inside it reads the register value and returns string with formatted value

```c
static ssize_t data_show(struct device *child, struct device_attribute *attr, char *buf)
{
	struct mmsens *dev = dev_get_drvdata(child);

	u32 data = ioread32(dev->base_addr + MMSENS_DATA_OFFSET);
	data &= DATA_MASK;

	return sprintf(buf, "%04X\n", data);
}
```

The details of the implementation are available in [github](https://github.com/straxy/mmsens-drv) repository.

# Building driver out-of-tree {#module}

The kernel driver can be provided in two ways: as part of the kernel source code, or as an out-of-tree entity. In first case, using the kernel configuration (`menuconfig` for instance) it can be selected whether driver will be built into the kernel, or as a separate kernel module (`.ko` extension). In the out-of-tree build, kernel header files are needed and driver can be built as a kernel module file.

In this case, we will be using the out-of-tree approach. If we have the kernel source code in the `$KERNEL_PATH`, the `Makefile` for building the kernel module would look like

```makefile
obj-m := mmsensdrv.o

SRC := $(shell pwd)

all:
	$(MAKE) ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -C $(KERNEL_PATH) M=$(SRC)
```

After executing `make` command, the `mmsensdrv.ko` file would be available and can be transferred to the SD card and tested.

# Adding driver to Yocto {#yocto}

If we want to add driver to Yocto a new recipe has to be created. The template (skeleton) exists at `poky/meta-skeleton/recipes-kernel/hello-mod` and we will reuse it.

The recipe for the module should look like

```bitbake
SUMMARY = "Memory-mapped QEMU sensor driver"
DESCRIPTION = "${SUMMARY}"
LICENSE = "GPLv2"
LIC_FILES_CHKSUM = "file://COPYING;md5=12f884d2ae1ff87c09e5b7ccc2c4ca7e"

inherit module

SRC_URI = "git://github.com/straxy/mmsens-drv.git;protocol=https;branch=main"
SRCREV = "${AUTOREV}"

S = "${WORKDIR}/git"

# The inherit of module.bbclass will automatically name module packages with
# "kernel-module-" prefix as required by the oe-core build environment.

RPROVIDES_${PN} += "kernel-module-mmsens-drv"

```

This module also needs to be added to the image recipe so it will be included in the output root filesystem image

```bitbake
IMAGE_INSTALL += "kernel-module-mmsens-drv"
```

After these changes have been added, the image can be rebuilt and run inside QEMU.

# Testing driver {#test}

Once Linux kernel is started inside QEMU, the module can be loaded. If `mmsensdrv.ko` is copied to the SD card manually, it can me loaded into the kernel with

```bash
$ insmod mmsensdrv.ko
```

If driver was included in Yocto image, it will be loaded automatically on boot.

First we can check that `mmsensX` file exists in the `/dev` directory

```bash
$ ls /dev/mmsens*
/dev/mmsens0
```

Reading that file should return value 0 since device must be started manually, by setting the `EN` bit in `CTRL` register.

```bash
$ cat /dev/mmsens0 
mistra.mmsens:DATA: read of value 0x0
0
```

Next, we can check that appropriate entries exist in the sysfs

```bash
$ ls /sys/class/mmsens/mmsens0/
available_frequencies  device                 frequency              subsystem
data                   enable                 interrupt              uevent
dev                    enable_interrupt       power
```

Finally, we can do the proper testing.

## Data incrementing

If we enable the device and try to read `data` attribute, as well as `/dev/mmsens0`, we should see that values are changing.

```bash
$ echo 1 > /sys/class/mmsens/mmsens0/enable
mistra.mmsens:CTRL: read of value 0x0
mistra.mmsens:CTRL: write of value 0x1
r_ctrl_post_write: Wrote 1 to CTRL
$ sleep 10
$ cat /sys/class/mmsens/mmsens0/data && cat /dev/mmsens0
mistra.mmsens:DATA: read of value 0x10
0010
mistra.mmsens:DATA: read of value 0x10
16
```

After enabling the device, the `data` attribute returns the BCD formatted value, while `/dev/mmsens0` returns the raw (integer) value, as expected.

## Frequency change

The list of available frequencies can be obrained from the `available_frequencies` attribute and current frequency selection from `frequency` attribute.

```bash
$ cat /sys/class/mmsens/mmsens0/available_frequencies
normal fast
$ cat /sys/class/mmsens/mmsens0/frequency 
mistra.mmsens:CTRL: read of value 0x0
normal
```

Per specification, `normal` frequency means that value changes once per second, while `fast` frequency means that value changes twice per second (every 0.5 seconds).

If we change sampling frequency from `normal` to `fast`, we should see that values are changing twice as often.

```bash
# Before
$ pushd /sys/class/mmsens/mmsens0
$ cat data && sleep 1 && cat data
0081
0082
# Change
$ echo fast > frequency
# After
$ cat data && sleep 1 && cat data
0085
0087
# Cleanup
$ popd
```

## Interrupt generation

Finally, we should check that interrupts are generated properly. However, since we do not have userspace application that would do something useful with those interrupts, we can use output from `/proc/interrupts`

```bash
# Before enabling interrupt generation
$ cat /proc/interrupts | grep mmsens
 40:          0     GIC-0  61 Level     10018000.mmsens
```

> _**NOTE**_: Where is value `29` from our description? First 32 interrupt lines are private interrupts per core, so numbering actually starts from 32. That means that actual interrupt number is 61 (32+29).

We can enable interrupt by writing 1 to `enable_interrupt` and number of occurences (column 1) increases. It is also visible from QEMU debug code that once interrupt is generated, interrupt handler in device driver clears `STATUS` register, thus acknowledging interrupt.

```bash
# After enabling interrupt generation
$ echo 1 > /sys/class/mmsens/mmsens0/enable_interrupt
mm_sens_update_irq: Interrupt generated
[ 8104.791017] Interrupt received
mistra.mmsens:STATUS: write of value 0x0
r_status_post_write: Wrote 0 to STATUS
mm_sens_update_irq: Interrupt none
$ cat /proc/interrupts | grep mmsens
 40:          1     GIC-0  61 Level     10018000.mmsens
```

# Summary {#summary}

In this blog post a simple character device platform driver is presented. The driver allows initialization and bit-field access of memory mapped peripheral.

Device Tree description for the memory-mapped device is also shown, which allows device drvier to obtain information about the device that is present.

The driver itself handles interrupt, but we have not gone into processing that event, and if someone would want to use it as it is (for instance with bash script), they could only do a polling approach to handle data.

<br />

Next step is to develop a user space application which will be able to initialize device using the driver, as well as receive information about received interrupt and process updated data. This will be done in [next blog post](#) in this series.
