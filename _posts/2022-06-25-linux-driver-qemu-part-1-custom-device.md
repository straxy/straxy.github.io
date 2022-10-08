---
layout: post
title: Developing custom memory-mapped peripheral for QEMU
tags: memory-mapped device-tree linux qemu u-boot
---

*This is part 1 of the Linux device driver development post series.*

In the [previous post](https://straxy.github.io/2022/05/19/linux-device-driver-development-qemu/) I presented some of the goals for doing this work with QEMU.

In this post I will cover the following things

- [Designing custom memory-mapped device in QEMU](#designing-custom-memory-mapped-device-in-qemu)
  - [Register map](#register-map)
  - [Fitting into Vexpress-A9 memory map](#fitting-into-vexpress-a9-memory-map)
- [QEMU implementation](#qemu-implementation)
  - [Register mapping](#register-mapping)
  - [QEMU timers](#qemu-timers)
  - [QEMU interrupt handling](#qemu-interrupt-handling)
- [Integrating with board file and build system](#integrating-with-board-file-and-build-system)
- [Testing developed peripheral](#testing-developed-peripheral)
- [Summary](#summary)

# Designing custom memory-mapped device in QEMU

QEMU can emulate many different peripherals. More importantly, it is possible to create new peripherals that are emulated.

That is particularly useful when someone wants to learn device driver development, since that peripheral will be unique, and it would be impossible to find already existing driver.

<br />

In this post we will develop a new, custom peripheral for QEMU.

The peripheral will have several 32-bit registers and ability to generate interrupts.
It will function as an 4-digit <abbr title="Binary-coded Decimal">BCD counter</abbr>. It can be enabled or disabled, generate interrupts on each count, and counting frequency can be selected.

In the next post we will develop Linux device driver that will enable access to this peripheral.

## Register map

Register map for the custom peripheral is shown in the following table

<table border="1" style="border: 1px solid black; width: 100%; border-collapse: collapse;">
  <thead>
    <tr>
      <th>offset</th>
      <th>name</th>
      <th>description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0x0</td>
      <td>`CONFIG`</td>
      <td>Configuration register, used to enable component, interrupt generation and select frequency</td>
    </tr>
    <tr>
      <td>0x4</td>
      <td>`STATUS`</td>
      <td>Status register, used to read/clear interrupt flag</td>
    </tr>
    <tr>
      <td>0x8</td>
      <td>`DATA`</td>
      <td>Data register holding current counter value</td>
    </tr>
  </tbody>
</table>

Bit values of `CONFIG` register are shown in the following table

<table border="1" style="border: 1px solid black; border-collapse: collapse; width: 100%;">
  <thead>
    <tr>
    <th style="width: 15%">name</th>
    <th>pos</th>
    <th>dflt</th>
    <th style="width: 60%">description</th>
  </tr>
  </thead>
  <tbody>
    <tr>
      <td><i>Reserved</i></td>
      <td>31:3</td>
      <td>0</td>
      <td>Reserved</td>
  	</tr>
    <tr>
      <td>`FREQ`</td>
      <td>2</td>
      <td>0</td>
      <td>Frequency setting:
      	<ul>
          <li>`0` - normal frequency (1 Hz)</li>
          <li>`1` - fast frequency (2 Hz)</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td>`IEN`</td>
      <td>1</td>
      <td>0</td>
      <td>Interrupt enable</td>
    </tr>
    <tr>
      <td>`EN`</td>
      <td>0</td>
      <td>0</td>
      <td>Enable device</td>
    </tr>
  </tbody>
</table>

Bit values of `STATUS` register are shown in the following table

<table border="1" style="border: 1px solid black; width: 100%; border-collapse: collapse;">
  <thead>
    <tr>
    <th style="width: 15%">name</th>
    <th>pos</th>
    <th>dflt</th>
    <th style="width: 60%">description</th>
  </tr> 
  </thead>
  <tbody>
    <tr>
      <td><i>Reserved</i></td>
      <td>31:2</td>
      <td>0</td>
      <td>Reserved</td>
  	</tr>
    <tr>
      <td>`IFG`</td>
      <td>1</td>
      <td>0</td>
      <td>Interrupt flag</td>
    </tr>
    <tr>
      <td><i>Reserved</i></td>
      <td>0</td>
      <td>0</td>
      <td>Reserved</td>
    </tr>
  </tbody>
</table>

Bit values of `DATA` register are shown in the following table

<table border="1" style="border: 1px solid black; width: 100%; border-collapse: collapse;">
  <thead>
    <tr>
    <th style="width: 15%">name</th>
    <th>pos</th>
    <th>dflt</th>
    <th style="width: 60%">description</th>
  </tr>
  </thead>
  <tbody>
    <tr>
      <td><i>Reserved</i></td>
      <td>31:16</td>
      <td>0</td>
      <td>Reserved</td>
  	</tr>
    <tr>
      <td>`SAMPLE`</td>
      <td>15:0</td>
      <td>0</td>
      <td>Current counter value</td>
    </tr>
  </tbody>
</table>

## Fitting into Vexpress-A9 memory map

In order to instantiate and use developed memory-mapped component, we need to integrate it into the Vexpress-A9 memory map and connect it to interrupt controller.

Looking at the [memory map](https://developer.arm.com/documentation/dui0447/j/programmers-model/memory-maps/arm-legacy-memory-map?lang=en) for Vexpress-A9, there are several regions that are unused, where we can place our custom component. In this example, I chose offset `0x00018000` in the motherboard peripheral memory map, region CS7. This means that the absolute address is `10018000`.

Also, we need an interrupt line for the custom component. Again, looking at the [documentation](https://developer.arm.com/documentation/dui0447/j/hardware-description/interrupt-signals?lang=en), there are several "reserved" interrupt lines which we can use. In this example, I chose interrupt line **29**.

# QEMU implementation

Details of implementation of a new memory-mapped device in QEMU will be shown in this section. Main points will be displayed, so someone can use it as instructions for creating a new device.

The device will be described by it's registers and added to the Versatile Express memory map.

## Register mapping

QEMU has a very simple and verbose way of describing registers of a component.

For each register, it's offset from the base address is specified, as well as bit-fields that the register consists of. Additionally, for each bitfiled access permissions can be specified and callback functions that are executed before and/or after accessing the register.

<br />

Register description for our [custom component](#designing-custom-memory-mapped-device-in-qemu) can be described as

```c
REG32(CTRL, 0x00)
    FIELD(CTRL,     EN,     0,  1)      /* component enable */
    FIELD(CTRL,     IEN,    1,  1)      /* interrupt enable */
    FIELD(CTRL,     FREQ,   2,  1)      /* sampling frequency setting */

REG32(STATUS, 0x04)
    FIELD(STATUS,   IFG,    1,  1)      /* interrupt flag */

REG32(DATA, 0x08)
    FIELD(DATA,     SAMPLE, 0,  16)     /* current value */
```

The previous code excerpt defines that we have three 32-bit registers called `CTRL` (offset 0x0 from base address), `STATUS` (offset 0x4) and `DATA` (offset 0x8). If we look at the the register `CTRL`, it has three bit-fields that are used: `EN` (bit position 0, size 1 bit), `IEN` (bit position 1, size 1 bit), `FREQ` (bit position 2, size 1 bit). Similar goes for other two registers.

> _**NOTE:**_ The details of the `REG32` and `FIELD` macros can be found in `hw/registerfields.h` include file. Thing to keep in mind is that these macros create additional macros (mask, shift, etc.) which we can use later for accessing individual registers and bit-fields.

<br />

In order to specify actions that are performed when someone tries to access these registers (read or write), following data must be defined

```c
static const RegisterAccessInfo mm_sens_regs_info[] = {
    {   .name = "CTRL",           .addr = A_CTRL,
        .reset = 0,
        .rsvd = ~(R_CTRL_EN_MASK | R_CTRL_IEN_MASK | R_CTRL_FREQ_MASK),
        .post_write = r_ctrl_post_write,
    },
    {   .name = "STATUS",           .addr = A_STATUS,
        .reset = 0,
        .rsvd = ~R_STATUS_IFG_MASK,
        .post_write = r_status_post_write,
    },
    {   .name = "DATA",         .addr = A_DATA,
        .reset = 0,
        .rsvd = ~R_DATA_SAMPLE_MASK,
        .ro = R_DATA_SAMPLE_MASK,
    },
};

static const MemoryRegionOps mm_sens_reg_ops = {
    .read = register_read_memory,
    .write = register_write_memory,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .valid = {
        .min_access_size = 4,
        .max_access_size = 4,
    }
};
```

Looking at the second structure, `mm_sens_reg_ops`, it defines that we are using register read and write functions when accessing this component.

Register functions use the `mm_sens_regs_info` array defined above, where `RegisterAccessInfo` structure has several useful fields:

* `name` - register name, used for debugging
* `addr` - register offset
* `reset` - reset value of register
* `ro` - read-only bitmask
* `w1c` - write one to clear bitmask
* `cor` - clear one read bitmask
* `rsvd` - reserved bits bitmask
* `pre_write` - callback executed before write command
* `post_write` - callback executed after write command
* `post_read` - callback executed after read command

The `post_write` functions in the previous code block need to perform certain actions based on the values that are written to the registers.

For instance, after `CTRL` register bit `EN` bit is set to 1, the `DATA` values should start incrementing. Or, after bit `FREQ` has changed, the frequency of incrementing `DATA` register should change.

Before we go into details of `post_write` functions, it would be useful to first explain how is `DATA` register periodically incremented, as well as how are interrupts implemented in QEMU code.

## QEMU timers

QEMU uses timers to enable periodical execution. Timers have a simple API (described in `hw/ptimer.h`) which we will go over in this section.

Our plan is to increment value of `DATA` register at two different frequencies: normal (1 Hz) and fast (2 Hz). The value should be incremented only when bit `EN` in `CTRL` register is set to 1. The value should also be incremented as BCD value, so each nibble should have values only in the range of 0-9.

<br />

Having analyzed our needs, following `ptimer` functions are of interest

* `ptimer_init` - create timer object and define callback that is executed when timer period expires
* `ptimer_set_freq` - set timer reload frequency in Hz
* `ptimer_run` - start timer and select whether continuous or oneshot mode is used
* `ptimer_stop` - stop timer

Based on this, the `post_write` function for `CTRL` has two parts.

In the first part `FREQ` is handled so after every write a check is made if value of `FREQ` bit has changed, and if it has, timer frequency is updated.

```c
// first part, FREQ handling
...
    new_sfreq = (s->regs[R_CTRL] & R_CTRL_FREQ_MASK) >> R_CTRL_FREQ_SHIFT;

    if (new_sfreq != s->sampling_frequency) {
        s->sampling_frequency = new_sfreq;
        switch (s->sampling_frequency) {
            case FREQ_NORMAL:
                ptimer_set_freq(s->timer, DATA_UPDATE_NORMAL_FREQ);
                break;
            case FREQ_FAST:
                ptimer_set_freq(s->timer, DATA_UPDATE_FAST_FREQ);
                break;
            default:
                DB_PRINT("Unknown frequency %u\n", s->sampling_frequency);
                break;
        }
    }
...
```

In the second part `EN` is handled and timer is started/stopped if `EN` bit is set/reset.

Additionally, if timer is enabled and `IEN` bit is also set, then evaluation of interrupt generation condition must be made (more on interrupts in next subsection).

```c
// second part, EN/IEN handling
...
    if (s->regs[R_CTRL] & R_CTRL_EN_MASK) {
        /* start timer if not started*/
        ptimer_run(s->timer, 0);

        if (s->regs[R_CTRL] & R_CTRL_IEN_MASK) {
            /* check if alarm should be triggered */
            mm_sens_update_irq(s);
        }
    } else {
        /* stop timer */
        ptimer_stop(s->timer);
    }
...
```

Increments of `DATA` register are implemented in the timer callback in the following manner

```c
// DATA incrementing
static void mm_sens_update_data(void *opaque)
{
    MMSensor *s = MM_SENS(opaque);

    s->regs[R_DATA] = s->regs[R_DATA] + 1;
    if ((s->regs[R_DATA] & 0x000fu) > 0x0009u) {
        s->regs[R_DATA] += 0x0006u;
        if ((s->regs[R_DATA] & 0x00f0u) > 0x0090u) {
            s->regs[R_DATA] += 0x0060u;
            if ((s->regs[R_DATA] & 0x0f00u) > 0x0900u) {
                s->regs[R_DATA] += 0x0600u;
                if ((s->regs[R_DATA] & 0xf000u) > 0x9000u) {
                    s->regs[R_DATA] += 0x6000u;
                }
            }
        }
    }

    s->regs[R_STATUS] |= R_STATUS_IFG_MASK;

    mm_sens_update_irq(s);
}
```

This way the BCD requirement is met and counting will look like in the following diagram

![BCD counting](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgCgpisXpci03il0veNqAdvm1GjZ8ZcJ0yD7KRmQe9Nwiz6Tu_VMUAceyDxppPyc3PD45VrO3yDGow5-AFHK3rFdapA3R3oZmHYn1EBexrT5obSbNWHCI8FkbGQ2QnFL1r4WeFEAV4nKySie3oIqjDhhdJBKDFlPgn0Os6Vgx9-MUirSuzyzzN26DYt/s720/bcd.png)

## QEMU interrupt handling

Interrupt handling in peripheral in QEMU is performed using the `qemu_set_irq` function. The function receives an additional parameter which indicates whether interrupt is pending or not. If interrupt is pending (and is not masked in the interrupt controller) it will be raised to the CPU.

In the case of our peripheral, interrupt is pending if both bit `IFG` in `STATUS` register and bit `IEN` in `CTRL` register are set. This condition has to be checked every time a change happens to any of these two registers, so there is a function that can be reused.

```c
// IRQ handling
static void mm_sens_update_irq(MMSensor *s)
{
    bool pending = s->regs[R_CTRL] & s->regs[R_STATUS] & R_CTRL_IEN_MASK;

    qemu_set_irq(s->irq, pending);
}
```

# Integrating with board file and build system

In order to use the custom memory mapped peripheral, it must be 'placed' in the memory map of the emulated board. Since we are using Versatile Express A9, then it's description must be updated.

Luckily, this is done with one simple command, where we can see chosen base address (`0x10018000`) and interrupt number (29).

```c
sysbus_create_simple("mistra.mmsens", 0x10018000, pic[29]);
```

Since build system uses meson and ninja, the new component file is added to the build system in the following manner

```python
softmmu_ss.add(files('mmsens.c'))
```

The patch file with implementation of the custom component is available in [github](https://github.com/straxy/qemu-custom-peripherals/blob/main/BCD-memory-mapped.patch). The main details of our custom component were explained in previous sections. However, there are standard QEMU structures that also must be used in order to describe VMState, as well as initialization and those can be reused from the patch.

Patch is applied to the QEMU source tree with following command

```bash
$ cd $QEMU_SRC
$ patch -p1 < BCD-memory-mapped.patch
```

After the patch is applied, QEMU must be rebuilt.

# Testing developed peripheral

Testing peripheral without appropriate Linux device driver is a bit harder, but not impossible.

We can use the embedded debug prints from the memory-mapped component. Before they can be used, the `MM_SENS_ERR_DEBUG` definition must be changed from `0` to `1`. This way all debug prints from the component will be visible.

<br />

U-Boot has integrated commands for reading and writing to memory addresses, so we can use it to try to enable the component, interrupt generation and read current data value.

After QEMU is started with

```bash
# Run QEMU with SD card and networking
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel $UBOOT \
                  -drive file=sd.img,format=raw,if=sd \
                  -net nic -net tap,ifname=qemu-tap0,script=no \
                  -serial mon:stdio
```

U-Boot prompt should be reached by pressing a key.

Following commands are available

* `md <addr>` - read data from address `addr`
* `mw <addr> <val>` - write `val` to address `addr`

We can first try reading the `CTRL` and `DATA` registers

```bash
# Run QEMU with SD card and networking
U-Boot> md 0x10018000 1
10018000:mistra.mmsens:CTRL: read of value 0x0
 00000000                               ....
U-Boot> md 0x10018008 1
10018008:mistra.mmsens:DATA: read of value 0x0
 00000000                               ....
```

The lines starting with `10018000:mistra.mmsens:` at debug prints from QEMU, while second lines are written by U-Boot.

In order to enable peripheral, so `DATA` value starts incrementing, we can write 1 to `EN` bit in `CTRL` register. If we read `DATA` register afterwards, we can see that the values are changing.

```bash
# Run QEMU with SD card and networking
U-Boot> mw 0x10018000 1
mistra.mmsens:CTRL: write of value 0x1
r_ctrl_post_write: Wrote 1 to CTRL
U-Boot> md 0x10018008 1
10018008:mistra.mmsens:DATA: read of value 0x1
 00000001                               ....
U-Boot> md 0x10018008 1
10018008:mistra.mmsens:DATA: read of value 0x2
 00000002                               ....
```

We can also check that the `IFG` in `STATUS` register is set. However, interrupt is not triggered since handling of this interrupt is not implemented in U-Boot, which is expected. We will implement interrupt handling in the next blog post, when we develop the Linux device driver for this component.

# Summary

In this blog post the process of developing a custom memory-mapped peripheral for QEMU is shown. Main details are described and the complete patch is available with full details of the component.

Using this process many different components can be implemented.

<br />

In the [next](https://straxy.github.io/2022/08/14/linux-driver-qemu-part-2-char-driver/) blog post I will show process of development of Linux device driver for this component.
