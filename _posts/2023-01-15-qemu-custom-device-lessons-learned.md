---
layout: post
title: QEMU Custom Devices - Lessons Learned
tags: qemu memory-mapped i2c
---

In the [Learning Linux device driver development using QEMU](https://straxy.github.io/2022/05/19/linux-device-driver-development-qemu/) blog series I showed examples of memory-mapped and I2C QEMU devices.

In the mean time I have been working on contributing to QEMU. I focused on Allwinner support, specifically on Allwinner A10 support. My idea was to use similar approach to the one implemented for OrangePi-PC and Allwinner-H3, where user can pass an SD card image and QEMU can perform the complete boot sequence.

The patches have been merged and review process is available here:
- [U-Boot support fix for SD card boot](https://patchew.org/QEMU/20221112214900.24152-1-strahinja.p.jankovic@gmail.com/)
- [U-Boot SPL support for SD card boot](https://patchew.org/QEMU/20221226220303.14420-1-strahinja.p.jankovic@gmail.com/)

<br/>

While working on these patches, the reviewers had very useful comments and suggestions. Based on those, I have also updated the patches used in the [Learning Linux device driver development using QEMU](https://straxy.github.io/2022/05/19/linux-device-driver-development-qemu/) blog series, to match the latest QEMU version and improve the coding standard. The updated patches are in the `qemu-7.2` branch of the [QEMU custom peripherals](https://github.com/straxy/qemu-custom-peripherals) repository.

<!--more-->

Items that will be covered in this post are

- [Init vs reset](#init-vs-reset)
- [Three-phase reset](#three-phase-reset)
- [Tracing instead of debug prints](#tracing-instead-of-debug-prints)
- [Avocado framework testing](#avocado-framework-testing)
- [Summary](#summary)

# Init vs reset

When device is initialized several callback functions are called in specific order:

- `.init()`
- `.realize()`
- `.reset()`

Therefore, if part of the initialization will be done in the `.reset()` function, it does not need to be duplicated in the `.init()` function.
For example, the initialization of the I2C component can be optimized so `.init()` function is removed, since the same initialization is done inside the `.reset()` function.

# Three-phase reset

The reset process for devices has been improved in recent versions of QEMU, so instead of single `.reset()` callback, there are multiple functions with different purpose:

- `.reset_enter()` - as name says, used when entering reset state and can modify only local state, i.e. should not cause any side-effects for other components;
- `.reset_hold()` - executed after `.reset_enter()` and can affect other components;
- `.reset_exit()` - executed before leaving reset state, can affect other components.

For more details, following [documentation](https://www.qemu.org/docs/master/devel/reset.html#multi-phase-mechanism) can be consulted.

In the simplest case, the old `.reset()` callback can be replaced with the `.reset_hold()` callback, which was used in both patches.

# Tracing instead of debug prints

This is one of the more useful features. The use of debug prints depends on macros, which require recompilation in order to enable or disable them.

With tracing, the support is compiled into the built binary file, and can be enabled or disabled just by passing a parameter `--trace`. Multiple macros can be enabled with multiple parameters, and they also accept wildcards, so only selected trace functions can be printed.

For example, if we want to print tracing data related to memory-mapped sensor, we can start QEMU with following parameters

```bash
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel u-boot.elf \
                  -drive file=sd.img,format=raw,if=sd -serial mon:stdio \
                  --trace "mm_sens*"
mm_sens_r_post_write CTRL: Wrote 0x0
mm_sens_r_post_write STATUS: Wrote 0x0
mm_sens_update_irq_none Interrupt none
[...]
```

Adding trace funcions is easy - every directory has the `trace-events` file where trace function protypes can be added for each file on that level. For instance, for memory-mapped sensor, the file would be `hw/misc/trace-events` and following funcions could be defined

```c
mm_sens_update_irq_generated(void) "Interrupt generated"
mm_sens_update_irq_none(void) "Interrupt none"
mm_sens_r_post_write(const char * reg, uint64_t val) "%s: Wrote 0x%"PRIx64
```

As shown, after the function prototype goes the format string which will be used for printing data. The code that should use trace logging should only call the appropriate function prefiexd with `trace_` and pass corresponding arguments. So in case of `mm_sens_post_write` the call to the trace function would be done as

```c
trace_mm_sens_r_post_write("CTRL", 0x01);
```

Other data types are also available and can be used. For more details [official documentation](https://qemu-project.gitlab.io/qemu/devel/tracing.html) can be used.

# Avocado framework testing

Avocado framework is used for QEMU integration testing. More details can be found in the [official documentation](https://github.com/qemu/qemu/blob/master/docs/devel/testing.rst#integration-tests-using-the-avocado-framework).

There are several QEMU helper classes which make instantion and running of test easier. Test is usually performed by avocado running a QEMU instance and performing some input over serial port and expecting defined output.

One simple avocado test could be to run the Yocto image that was built in the [QEMU Board Emulation Part 5](https://straxy.github.io/2022/11/05/qemu-board-emulation-part-5-vexpress-yocto-qt6/) and wait until login prompt is shown. Test design is shown in the following code excerpt, and the only files that are required are the `u-boot.elf` and `rootfs.wic.gz` (using `.gz` so file size is only 26 MiB).

```python
def test_arm_vexpress_yocto(self):
    """
    :avocado: tags=arch:arm
    :avocado: tags=machine:vexpress-a9
    :avocado: tags=mistra
    """
    uboot_url = ("file:///home/user/u-boot.elf")
    uboot_hash = '7571e33fcb1df23b05d31ae1463f0bcf9250eee2'
    uboot_path = self.fetch_asset(uboot_url, asset_hash=uboot_hash)

    sdcard_url = ('file:///home/user/rootfs.wic.gz')
    sdcard_hash = '15e73cb70094fd0d4b6d933664c9e9ff07353dae'
    sdcard_path_gz = self.fetch_asset(sdcard_url, asset_hash=sdcard_hash)
    sdcard_path = os.path.join(self.workdir, 'sd.img')
    archive.gzip_uncompress(sdcard_path_gz, sdcard_path)
    image_pow2ceil_expand(sdcard_path)

    self.vm.set_console()
    self.vm.add_args('-m', '1G',
                        '-kernel', uboot_path,
                        '-drive', 'file=' + sdcard_path + ',if=sd,format=raw')
    self.vm.launch()
    console_pattern = 'Mistra FrameBuffer 3.1'
    self.wait_for_console_pattern(console_pattern)
```

The avocado test can be run using the following command from the directory where QEMU is build (`qemu/bin/arm` if instructions from the blog post series are followed)

```bash
$ avocado --show=app,console run -t mistra tests/avocado/boot_linux_console.py
 (1/1) tests/avocado/boot_linux_console.py:BootLinuxConsole.test_arm_vexpress_yocto: STARTED
1-tests/avocado/boot_linux_console.py:BootLinuxConsole.test_arm_vexpress_yocto: console: U-Boot 2021.04 (Apr 05 2021 - 15:03:29 +0000)
1-tests/avocado/boot_linux_console.py:BootLinuxConsole.test_arm_vexpress_yocto: console: DRAM:  1 GiB

[...]

1-tests/avocado/boot_linux_console.py:BootLinuxConsole.test_arm_vexpress_yocto: console: [   19.490599] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
1-tests/avocado/boot_linux_console.py:BootLinuxConsole.test_arm_vexpress_yocto: console: Mistra FrameBuffer 3.1 vexpress-qemu /dev/ttyAMA0
 (1/1) tests/avocado/boot_linux_console.py:BootLinuxConsole.test_arm_vexpress_yocto: PASS (30.37 s)
RESULTS    : PASS 1 | ERROR 0 | FAIL 0 | SKIP 0 | WARN 0 | INTERRUPT 0 | CANCEL 0
```

The `-t mistra` is the tag selector. Another way to run this test would have been `-t machine=vexpress-a9` or `-t arch=arm`, but these would also run all other tests with those tags.

# Summary

The process of submitting patches and updating them accoring the review comments for the QEMU was interesting and I learned some new things. I am very grateful to the QEMU community since this was a very pleasant experience.

I would definitely suggest everyone to try contributing to an open-source project, since it can be beneficial for everyone - both the project and the developers.
