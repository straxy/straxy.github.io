---
layout: post
title: Custom I2C QEMU peripheral and userspace application
tags: i2c yocto userspace cmake cpp linux qemu
---

*This is part 4 of the Linux device driver development post series.*

After showing custom memory-mapped sensor and it's handing in previous posts, we will now take a look at I2C peripheral.

In this post we will cover the following things

- [Custom I2C QEMU peripheral](#custom-i2c-qemu-peripheral)
  - [Peripheral description](#peripheral-description)
  - [Peripheral implementation](#peripheral-implementation)
    - [Writing to the device](#writing-to-the-device)
    - [Reading from device](#reading-from-device)
  - [Integrating into QEMU](#integrating-into-qemu)
- [I2C Userspace handling](#i2c-userspace-handling)
  - [Initialization](#initialization)
  - [Periodic execution](#periodic-execution)
- [Testing in QEMU](#testing-in-qemu)
  - [i2c-tools](#i2c-tools)
  - [Userspace application](#userspace-application)
- [Summary](#summary)

# Custom I2C QEMU peripheral

Emulating I2C peripheral under QEMU is not much harder then emulating a memory-mapped peripheral.

The peripheral needs to be connected to an existing I2C controller, assigned and I2C address and should provide responses when data is written to or read from I2C bus.

## Peripheral description

In this example we will implement a simple I2C temperature sensor. Once enabled, it will return a random value in the range of 15.0 to 25.0 with 0.5 degrees (Celsius) step on every read.


<br />

The peripheral has following registers

<table border="1" style="border: 1px solid black; width: 100%; border-collapse: collapse;">
  <thead>
    <tr>
      <th style="width: 15%">offset</th>
      <th>name</th>
      <th>description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0x0</td>
      <td><code>ID</code></td>
      <td>Unique ID register, always returning value <code>0x5A</code></td>
    </tr>
    <tr>
      <td>0x1</td>
      <td><code>CONFIG</code></td>
      <td>Configuration register, used to enable temperature temperature</td>
    </tr>
    <tr>
      <td>0x2</td>
      <td><code>TEMPERATURE</code></td>
      <td>Data register holding current temperature. The value is coded as <code>Q5.1</code> to be able to hold 0.5 degree step</td>
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
      <td>7:1</td>
      <td>0</td>
      <td>Reserved</td>
  	</tr>
    <tr>
      <td><code>EN</code></td>
      <td>0</td>
      <td>0</td>
      <td>Enable device</td>
    </tr>
  </tbody>
</table>

## Peripheral implementation

The implementation of the peripheral is available in [github](#) repository.

Functions of interest are the ones related to data handling. The global structure is defined in the following way

```c
typedef struct I2CSensor {
  /*< private >*/
  I2CSlave i2c;
  /*< public >*/
  uint8_t regs[NR_REGS];  // peripheral registers
  uint8_t count;          // counter used for tx/rx
  uint8_t ptr;            // current register index
} I2CSensor;
```

### Writing to the device

The only register that supports writing is the `CTRL` register.

In order to set the register that will be written, first I2C write must be the register address (that will update the `ptr` field), and second write must be the value that should be written.

```c
static int i2c_sens_tx(I2CSlave *i2c, uint8_t data)
{
  I2CSensor *s = I2C_SENS(i2c);

  if (s->count == 0) {
    /* store register address */
    s->ptr = data;
    s->count++;
  } else {
    if (s->ptr == REG_CTRL_OFFSET) {
      s->regs[s->ptr++] = data;
    }
  }
  return 0;
}
```
### Reading from device

Reading process is similar to writing: first one byte must be written to set the address of register to be read, and then the reading can proceed.

All registers support reading so it is just a matter of returning current register value.

```c
/* Called when master requests read */
static uint8_t i2c_sens_rx(I2CSlave *i2c)
{
  I2CSensor *s = I2C_SENS(i2c);
  uint8_t ret = 0xff;

  if (s->ptr < NR_REGS) {
    ret = s->regs[s->ptr++];
  }

  return ret;
}
```

We also want each read to trigger loading of random value to the `TEMPERATURE` register. That is achieved by defining an `event` callback which modifies the `s->regs[2]` value when `I2C_START_RECV` event is received.

The register is modified only if device is currently enabled, otherwise a value of `0xff` will be stored.

```c
static int i2c_sens_event(I2CSlave *i2c, enum i2c_event event)
{
  I2CSensor *s = I2C_SENS(i2c);

  if (event == I2C_START_RECV) {
    if (s->ptr == REG_TEMPERATURE_OFFSET) {
      if (s->regs[REG_CTRL_OFFSET] & REG_CTRL_EN_MASK) {
        s->regs[REG_TEMPERATURE_OFFSET] = i2c_sens_get_temperature();
      } else {
        s->regs[REG_TEMPERATURE_OFFSET] = 0xff;
      }
    }
  }

  s->count = 0;

  return 0;
}
```

## Integrating into QEMU

Similarly to the Memory-mapped peripheral, the new I2C peripheral must be integrated into the Versatile Express description. This time, instead of attaching to a place in memory map, it needs to be attached to an I2C controller at a selected I2C address.

We can choose either to use an already existing I2C controller, or to also add a new I2C controller in the memory map, and then attach this component to the I2C controller.

In this example we will add a new I2C controller at address `0x10010000` and attach our custom I2C component to that I2C controller. The I2C slave address is chosen as `0x36`.

The excerpt from initialization of emulated Versatile Express board is

```c
  dev = sysbus_create_simple(TYPE_VERSATILE_I2C, map[VE_I2C_SENSOR], NULL);
  i2c = (I2CBus *)qdev_get_child_bus(dev, "i2c");
  i2c_slave_create_simple(i2c, "mistra.i2csens", 0x36);
```

# I2C Userspace handling

With the device created in QEMU we can turn to making a userspace application to access the I2C device.

The program flow will be similar to the one used for memory-mapped device

1. Device should be initialized
2. After device is initialized, we should periodically read the temperature data

The full userspace application can be found in [github](https://github.com/straxy/i2csens-app).

## Initialization

The device initialization now involves setting up the I2C userspace communication. Everything is done using the `dev` file for the selected I2C controller. In our case it is `/dev/i2c-2`.

The process can be summarized as

1. Opening the I2C controller device file
   ```c
   fd = open("/dev/i2c-2", O_RDWR);
   ```
2. Configuring I2C slave address
   ```c
   ioctl(fd, I2C_SLAVE, 0x36);
   ```
3. Enabling device by writing `1` to `CTRL` register. Since this write should be performed in two steps, an array consisting of `CTRL` register offset (`0x01`) and value to be written (`1`) is passed to the `write` function
   ```c
   uint8_t buffer[2] = { 0x01, 0x01 };
   write(fd, buffer, 2);
   ```

## Periodic execution

After the device is initialized, we can use a separate thread to periodically initiate reads from I2C device. Unlike the memory-mapped device, this peripheral has no interrupt generation possibilities, so we need to make periodic execution in some other way.

We will use `std::this_thread::sleep_until` to enable periodic readouts from the I2C peripheral.

The read process also consists of two steps: writing `TEMPERATURE` register offset and then reading the `TEMPERATURE` register value.

```c
while (m_running)
{
  // Read current time so we know when to timeout
  current_time = std::chrono::system_clock::now();

  // read I2C device and print
  // first write address
  uint8_t buffer[1] = { 0x02 };
  write(fd, buffer, 1);
  // then read value
  read(fd, buffer, 1);

  std::cout << "Measured " << (buffer[0] / 2.) << std::endl;

  // sleep_until
  std::this_thread::sleep_until(current_time + std::chrono::seconds(1));
}
```

# Testing in QEMU

## i2c-tools

Before testing the userspace application, we need to test that device is integrated properly inside QEMU. For that we can use `i2c-tools` package (add it Yocto or install in Ubuntu rootfs).

Once `i2c-tools` are installed, we can use `i2cdetect` to verify that a peripheral at `0x36` address exists using

```bash
$ i2cdetect -y 2
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- 36 -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- 
```

The `i2cget` can be used to read a value and `i2cset` to set certain value.

We can check that `ID` register returns `0x5A`

```bash
$ i2cget -y 2 0x36 0
0x5a
```

The last test would be to turn on the device and check that `TEMPERATURE` register returns different values every time

```bash
$ i2cset -y 2 0x36 1 1
$ i2cget -y 2 0x36 2
0x22
$ i2cget -y 2 0x36 2
0x25
$ i2cget -y 2 0x36 2
0x2c
```

## Userspace application

Userspace application can be compiled in the same manner as the [memory-mapped application](https://straxy.github.io/2022/10/08/linux-driver-qemu-part-3-userspace-application/#building-application-manually), via external toolchain or using a Yocto SDK generated toolchain:

```bash
$ mkdir build && cd build
$ cmake -DCMAKE_TOOLCHAIN_FILE=../cmake/toolchain.cmake ..
$ make -j
```

or

```bash
$ source /opt/mistra-framebuffer/3.1/environment-setup-armv7at2hf-neon-mistra-linux-gnueabi
$ mkdir build && cd build
$ cmake ..
$ make -j
```

Alternatively, application can be added to the Yocto build system with a recipe shown in [github](https://github.com/straxy/meta-vexpress-apps/blob/main/recipes-apps/i2csens-app/i2csens-app_1.0.bb).

<br />

After the application has been built and copied to the rootfs, it can be started. The application will initialize the I2C peripheral and start printing read temperature value every second.

![I2C test](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEh3ZNCSU0Wu222PtxT9TsvJF_Z5H5JfBck64yXS3eKPSCuPEXyYa3cPKHTb01Iua591Rn_OI_5e4F8nAFVEqIALuBygPPh7j9PGdSsINI3sSX20wn1wj689WDGVcFkMkjwIlcXftw1kFEpWVxNF3QEMzMIOokbsBp-kR3Fyhxt_QcgoeKk1aHgfeoQv/s1600/Screen%20recording%202022-10-18%2022.11.10.gif")


# Summary

In this blog post userspace application development for custom QEMU I2C sensor is presented.

The application encapsulates the I2C userspace access and implements periodic reading of data.

It can be improved with unit tests, as well as more interactive behavior, but it is left for some other time.
