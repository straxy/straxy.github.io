---
layout: post
title: Running Vexpress board under QEMU with Ubuntu root filesystem
tags: ubuntu linux qemu u-boot
---

*This is part 2 of the QEMU Board Emulation post series.*

In the [previous post](https://straxy.github.io/2021/10/09/qemu-board-emulation-part-1-basics/) the basic steps for obtaining and compiling QEMU, U-Boot and Linux were presented. The only part that was missing for the complete system setup was the root filesystem. Also, all of the images were injected directly into emulated RAM memory using the QEMU's `loader` mechanism.


In this post I will cover the following things

- [Obtaining Ubuntu root filesystem](#obtaining-ubuntu-root-filesystem)
  - [Download Ubuntu root filesystem](#download-ubuntu-root-filesystem)
- [Booting and running from SD card](#booting-and-running-from-sd-card)
  - [Preparing SD card image](#preparing-sd-card-image)
    - [Creating an empty SD card image](#creating-an-empty-sd-card-image)
    - [Partitioning SD card image](#partitioning-sd-card-image)
    - [Formatting partitions](#formatting-partitions)
  - [Copying data to SD card image](#copying-data-to-sd-card-image)
    - [Boot partition](#boot-partition)
    - [Root filesystem partition](#root-filesystem-partition)
  - [Running QEMU with SD card image](#running-qemu-with-sd-card-image)
- [Using TFTP to boot and running from NFS root filesystem](#using-tftp-to-boot-and-running-from-nfs-root-filesystem)
  - [TFTP server](#tftp-server)
    - [Set up TFTP server](#set-up-tftp-server)
  - [NFS root filesystem](#nfs-root-filesystem)
    - [Set up NFS server](#set-up-nfs-server)
  - [Running QEMU with TFTP and NFS server](#running-qemu-with-tftp-and-nfs-server)
    - [Enable network in QEMU](#enable-network-in-qemu)
    - [Run QEMU with network](#run-qemu-with-network)
    - [Enable internet access in QEMU with NAT and tap](#enable-internet-access-in-qemu-with-nat-and-tap)
- [Github helper scripts](#github-helper-scripts)
- [Summary](#summary)

# Obtaining Ubuntu root filesystem

There are several ways to obtain root filesystem for an embedded system

* Prebuilt root filesystem - [Ubuntu Base](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/), [Armbian](https://www.armbian.com/)
* Manual custom-built root filesystem using [Busybox](https://busybox.net/)
* Guided/managed custom-build root filesystem using [Buildroot](https://buildroot.org/) or [Yocto](https://www.yoctoproject.org/)

In this post we will use the prebuilt Ubuntu root filesystem. In some of the future posts the Buildroot and Yocto approaches will be covered.

## Download Ubuntu root filesystem

The archive with the Ubuntu minimal 20.04 root filesystem can be obtained using the following step

```bash
# Prepare Ubuntu
$ wget -c https://rcn-ee.net/rootfs/eewiki/minfs/ubuntu-20.04.3-minimal-armhf-2021-12-20.tar.xz
```

Now that we have the Ubuntu root filesystem, it needs to be supplied to the Linux running under QEMU. Two options are to create a SD card image or to access it over network, and both will be covered in the following sections.

# Booting and running from SD card

QEMU supports emulation of the SD card interface. Depending on the board that is emulated, different SD card interfaces are available.

## Preparing SD card image

QEMU provides tool for creating the emulated SD card, `qemu-img`. Before the tool can be used, the environment script created in the [previous post](https://straxy.github.io/2021/10/09/qemu-board-emulation-part-1-basics/) needs to be sourced.

### Creating an empty SD card image

The SD card image can be created using the following command:

```bash
# Create empty SD card
$ cd $PROJ_DIR
$ qemu-img create sd.img 4G
Formatting 'sd.img', fmt=raw size=4294967296
```

The last parameter that is passed is the size and for this work size of 4GB is selected. After executing the previous command, file `sd.img` will be created.

Before the SD card can be used to copy data, it has to be partitioned and formatted.

In order to simplify further work, a new line can be added to the `$PROJ_DIR/env.sh` with the path to the SD card

```bash
# env.sh update
export SD_IMG=$PROJ_DIR/sd.img
```

### Partitioning SD card image

The SD card will be partitioned into two partitions.

The first one will be used for the kernel image and device tree files. The size will be 64MB and if will be later formatted as FAT32.

The second partition will take up the rest of the SD card and it will later be formatted as ext4.

We can use `fdisk` to check the status before and after partitioning:

```bash
$ fdisk -l ./sd.img
Disk ./sd.img: 4 GiB, 4294967296 bytes, 8388608 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

For formatting the SD card image we will use the `sfdisk` application.

Partitioning is done using the following command

```bash
# Partitioning the SD card
$ sfdisk ./sd.img << EOF
,64M,c,*
,,L,
EOF

Disk ./sd.img: 4 GiB, 4294967296 bytes, 8388608 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

>>> Created a new DOS disklabel with disk identifier 0x87facb0e.
./sd.img1: Created a new partition 1 of type 'W95 FAT32 (LBA)' and of size 64 MiB.
./sd.img2: Created a new partition 2 of type 'Linux' and of size 3,9 GiB.
./sd.img3: Done.

New situation:
Disklabel type: dos
Disk identifier: 0x87facb0e

Device     Boot  Start     End Sectors  Size Id Type
./sd.img1  *      2048  133119  131072   64M  c W95 FAT32 (LBA)
./sd.img2       133120 8388607 8255488  3,9G 83 Linux

The partition table has been altered.
Syncing disks.
```

The format of the `sfdisk` partitioning is `start,size,type,bootable`, meaning we are creating

- first partition, from the beggining of the card, size 64MB, type `c` (FAT32) and bootable
- second partition, that starts after the first partition, which will fill the available space, type `L` (ext4).

### Formatting partitions

After the SD card has been partitioned, and before the partitions can be formated using the `mkfs` application, the SD card must be "plugged in", i.e. the partitions must be recognized by the host operating system. This is done using the `kpartx` tool

```bash
# 'Inserting' the SD card
$ sudo kpartx -av ./sd.img
add map loop12p1 (253:0): 0 131072 linear 7:12 2048
add map loop12p2 (253:1): 0 8255488 linear 7:12 133120
```

The value 12 in the output loop**12**p1 can differ from system to system and that is why we are using the `-v` switch, so the value is printed. After this command, the partitions are visible in the system under `/dev/mapper/loop12p1` and `/dev/mapper/loop12p2`.

In order to format partitions, following commands will be used

```bash
# Formatting SD card partitions
$ sudo mkfs.vfat -F 32 -n "boot" /dev/mapper/loop12p1
$ sudo mkfs.ext4 -L rootfs /dev/mapper/loop12p2
```

## Copying data to SD card image

After the partitions have been formatted, the data can be copied. In order to copy data, the partitions need to be mounted.

The `/run/mount/` will be used as base for the mount points, where `boot` and `rootfs` directories will be created.

### Boot partition

The boot partition is mounted in the following way

```bash
# Mounting boot partition
$ sudo mkdir -p /run/mount/boot
$ sudo mount /dev/mapper/loop12p1 /run/mount/boot
```

Linux kernel `zImage` file and Device tree file need to be copied to the boot partition

```bash
# Copying files to boot partition
$ sudo cp $ZIMAGE /run/mount/boot
$ sudo cp $DTB /run/mount/boot
```

After data has been copied, the `boot` partition can be unmounted using

```bash
# Umount
$ sudo umount /run/mount/boot
```

### Root filesystem partition

The rootfs partition is mounted in the following way

```bash
# Mounting rootfs partition
$ sudo mkdir -p /run/mount/rootfs
$ sudo mount /dev/mapper/loop12p2 /run/mount/rootfs
```

Ubuntu rootfs needs to be unpacked and copied to the rootfs partition

```bash
# Copying Ubuntu files to boot partition
$ tar xf ubuntu-20.04.3-minimal-armhf-2021-12-20.tar.xz
$ sudo tar xfvp ./ubuntu-20.04.3-minimal-armhf-2021-12-20/armhf-rootfs-ubuntu-focal.tar -C /run/mount/rootfs/
```

Also, kernel modules must be installed

```bash
# Copying kernel modules and setting permissions
$ cd $PROJ_DIR/linux/build_vexpress
$ sudo make ARCH=arm INSTALL_MOD_PATH=/run/mount/rootfs modules_install
$ sync
```

After data has been copied, the `rootfs` partition can be unmounted using

```bash
# Umount
$ sudo umount /run/mount/rootfs
```

Now the SD card can be "unplugged" from the system using

```bash
# Unplug SD card
$ sudo kpartx -d $PROJ_DIR/sd.img
```

## Running QEMU with SD card image

After the SD card is ready, the QEMU can be started using the following command

```bash
# Run QEMU with SD card
$ cd $PROJ_DIR
$ qemu-system-arm -M vexpress-a9 -m 1G -kernel $UBOOT -nographic \
                  -drive file=sd.img,format=raw,if=sd
```

Once U-Boot starts, it can be used to copy kernel image and device tree file into RAM memory, as well as to set up the linux kernel command line

```bash
# Load items into memory and start kernel
u-boot> fatload mmc 0:1 0x62000000 zImage
u-boot> fatload mmc 0:1 0x68000000 vexpress-v2p-ca9.dtb
u-boot> setenv bootargs "console=ttyAMA0 root=/dev/mmcblk0p2 rw"
u-boot> bootz 0x62000000 - 0x68000000
```

After kernel boots the login prompt appears. User name is `ubuntu`, password `temppwd`

```bash
# Logged in Ubuntu
...
Ubuntu 20.04 LTS arm ttyAMA0

default username:password is [ubuntu:temppwd]

arm login:
```

![Booting Linux with Ubuntu rootfs](https://blogger.googleusercontent.com/img/a/AVvXsEjdlRr_Hm3805hS5C-BmVmQPl9bdyVp1nOC621Yaod7Nf6hZxXqIBlJLfOfSlnoiyH2O_qn18Nf6gW5ffwdoqDNsQX2dPfm7lK_mQX_UmVlyvjYrDJGKQepflz_-5zZBi10UHEbS6QBKqN65TwySOX2OJeJWOjIsCuzCcef9Cq4VK33cpXyXwQW7wve=s16000)

# Using TFTP to boot and running from NFS root filesystem

If the board has a network connection, then kernel files can be loaded from a remote location. Also, root filesystem can be accessed from a remote location.

QEMU emulates ethernet access, so it can be used for emulating the network boot.

## TFTP server

TFTP (Trivial File Transfer Protocol) is a protocol which allows files to be obtained from a remote server. In this case, we will use it to obtain the compressed kernel image and device tree blob. U-Boot has an integrated TFTP client which will be used to load those files into RAM memory.

### Set up TFTP server

In order to set up the TFTP server, it needs to be installed using the following command

```bash
# Install TFTP server
$ sudo apt install tftpd-hpa
```

Previous command will set up location `/srv/tftp` as the location from where the files can be downloaded remotely, so the `zImage` and `vexpress-v2p-ca9.dtb` files need to be copied into that directory.

```bash
# Copy files to the TFTP server.
$ sudo cp $ZIMAGE $DTB /srv/tftp/
```

## NFS root filesystem

NFS (Network File System) is a file system that can be accessed over network as if it were physically present. This can be very useful during application development, since application binary files can be copied directly to a directory on host system, and they will be available in the target system.

### Set up NFS server

In order to set up the NFS server, following needs to be installed

```bash
# Install NFS server
$ sudo apt install nfs-kernel-server
```

The directory in the host system that will be available to the target system needs to be configured by adding a line in the `/etc/exports` file (if the file does not exist, it should be created).

```bash
# /etc/exports addition
/home/user/rootfs *(rw,sync,no_subtree_check,no_root_squash)
```

In this example, the directory where NFS rootfs is located in the host system is `/home/user/rootfs`.
After the line has been added, the exports information should be updated using

```bash
# Reload exportfs information
$ sudo exportfs -rav
```

The root filesystem contents should be copied into the exported directory. We will be using the same Ubuntu root filesystem, so the archive can be extracted directly into the directory. Also, kernel modules must be installed and correct permissions must be set on the filesystem.

```bash
# Extract root filesystem and set permissions
$ sudo tar xfvp ./ubuntu-20.04.3-minimal-armhf-2021-12-20/armhf-rootfs-ubuntu-focal.tar -C /home/user/rootfs/
$ cd $PROJ_DIR/linux/build_vexpress
$ sudo make ARCH=arm INSTALL_MOD_PATH=/home/user/rootfs modules_install
$ sync
```

## Running QEMU with TFTP and NFS server

Before QEMU can emulate the TFTP booting and NFS root filesystem, network must be configured. QEMU, by default, creates a network connection to host machine. However, that network has limitations where emulated system can access outside network, but it is not accessible from the host system.

Besides the default network connection, QEMU supports different methods for enabling network access where emulated system is accessible from host system:

* using `tap` interface,
* using Bridged adapter network.

Both methods require manual setup before QEMU is started. The `tap` interface method is simpler, but the Bridged adapter network can make the emulated system accessible from rest of the network also, not only the host system.

In this example, we will use the `tap` interface to enable network connection.

More details about QEMU networking support can be found [here](https://wiki.qemu.org/Documentation/Networking).

### Enable network in QEMU

The `tap` interface can be configured in the following way

```bash
# Create tap interface
$ sudo tunctl -u $(whoami) -t qemu-tap0
$ sudo ifconfig qemu-tap0 192.168.123.1
$ sudo route add -net 192.168.123.0 netmask 255.255.255.0 dev qemu-tap0
$ sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```

With commands above the QEMU instance will be able to ping and access host computer, and other way around, but it will not be able to access internet. In order to enable internet access, following commands are required on the host (set `<interface>` to network interface that is used on host machine for accessing internet)

```bash
# Enable guest internet access
$ iptables -t nat -A POSTROUTING -o <interface> -j MASQUERADE
$ iptables -I FORWARD 1 -i qemu-tap0 -j ACCEPT
$ iptables -I FORWARD 1 -o qemu-tap0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

> _**NOTE**_: These commands need to be executed every time the host system is rebooted.

The commands for NAT networking were adapted from [here](https://felipec.wordpress.com/2009/12/27/setting-up-qemu-with-a-nat/). The approach with the bridged networking is a bit more complex and can be found [here](https://blog.elastocloud.org/2015/07/qemukvm-bridged-network-with-tap.html).

After the `tap` interface has been created the QEMU can be started.

### Run QEMU with network

The QEMU with networking can be started in the following way:

```bash
# Start QEMU with networking
$ qemu-system-arm -M vexpress-a9 -m 1G \
                  -kernel $UBOOT -nographic \
                  -net nic -net tap,ifname=qemu-tap0,script=no
```

After the U-Boot is started, the TFTP protocol can be used to copy Linux kernel and Device tree files into RAM memory.

```bash
# Load Linux kernel image and Device Tree file to RAM
u-boot> setenv serverip 192.168.123.1
u-boot> setenv ipaddr 192.168.123.101
u-boot> tftp 62000000 zImage                                                                               
smc911x: MAC 52:54:00:12:34:56
smc911x: detected LAN9118 controller
smc911x: phy initialized
smc911x: MAC 52:54:00:12:34:56
Using smc911x-0 device
TFTP from server 192.168.123.1; our IP address is 192.168.123.101
Filename 'zImage'.
Load address: 0x62000000
Loading: #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         ###########
         9.4 MiB/s
done
Bytes transferred = 9695744 (93f200 hex)
smc911x: MAC 52:54:00:12:34:56
u-boot&gt; tftp 68000000 vexpress-v2p-ca9.dtb
smc911x: MAC 52:54:00:12:34:56
smc911x: detected LAN9118 controller
smc911x: phy initialized
smc911x: MAC 52:54:00:12:34:56
Using smc911x-0 device
TFTP from server 192.168.123.1; our IP address is 192.168.123.101
Filename 'vexpress-v2p-ca9.dtb'.
Load address: 0x68000000
Loading: #
         1.4 MiB/s
done
Bytes transferred = 14173 (375d hex)
smc911x: MAC 52:54:00:12:34:56
```

Before the system can be started, the rootfs parameters must be configured so NFS is used:

```bash
# Use NFS as rootfs
u-boot> setenv npath /home/user/rootfs
u-boot> setenv bootargs "console=ttyAMA0 root=/dev/nfs rw nfsroot=${serverip}:${npath},tcp,v3 ip=${ipaddr}"
u-boot> bootz 62000000 - 68000000
```

Once the system is started, it will use NFS as rootfs, which can be checked

```bash
# Check NFS rootfs
...
[    3.351463] VFS: Mounted root (nfs filesystem) on device 0:15.
...
$ df -h
Filesystem                          Size  Used Avail Use% Mounted on
192.168.123.1:/home/user/rootfs     234G   83G  139G  38% /
devtmpfs                            464M     0  464M   0% /dev
tmpfs                               497M     0  497M   0% /dev/shm
tmpfs                               100M  2.8M   97M   3% /run
tmpfs                               5.0M     0  5.0M   0% /run/lock
tmpfs                               497M     0  497M   0% /sys/fs/cgroup
tmpfs                               100M     0  100M   0% /run/user/1000
```

The system can also be ping'ed from host

```bash
# Ping from host
host$ ping 192.168.123.101 -c 4
PING 192.168.123.101 (192.168.123.101) 56(84) bytes of data.
64 bytes from 192.168.123.101: icmp_seq=1 ttl=64 time=0.887 ms
64 bytes from 192.168.123.101: icmp_seq=2 ttl=64 time=0.726 ms
64 bytes from 192.168.123.101: icmp_seq=3 ttl=64 time=0.735 ms
64 bytes from 192.168.123.101: icmp_seq=4 ttl=64 time=0.743 ms

--- 192.168.123.101 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3074ms
rtt min/avg/max/mdev = 0.726/0.772/0.887/0.066 ms
```

![NFS boot](https://blogger.googleusercontent.com/img/a/AVvXsEjTD5kTvieOSWfs74eKW9ZMWoPvir0c31AEFNEELECgGsIQ7L7K8GVOWhUDUqhTCtoY0MCL3RxmXf3j5hM48-C90peQIJfLuosDC4pO5w0Ze3VL9i94yeIqDtRnBtqBHYyBgAecP5fotjDBldNV5OPDpTcnov4UF1_mAwnJrCCLtYDyjtJxe22Vmct0=s16000)

### Enable internet access in QEMU with NAT and tap

In order to enable internet access, two things need to be done in QEMU guest: set default gateway and update DNS server.

Setting default gateway can be done in the following way

```bash
# Set default gateway
$ sudo route add default gw 192.168.123.1
```

In order to set the DNS server, the `/etc/resolv.conf` file needs to be modified by adding the following line (use google server)

```bash
# Set DNS server
nameserver 8.8.8.8
```

After these changes are made, QEMU guest can access external network which can be simply verified

```bash
# Ping www.google.com
$ ping www.google.com -c 4
PING www.google.com (142.250.74.36) 56(84) bytes of data.
64 bytes from arn09s22-in-f4.1e100.net (142.250.74.36): icmp_seq=1 ttl=53 time=9.57 ms
64 bytes from arn09s22-in-f4.1e100.net (142.250.74.36): icmp_seq=2 ttl=53 time=6.89 ms
64 bytes from arn09s22-in-f4.1e100.net (142.250.74.36): icmp_seq=3 ttl=53 time=6.95 ms
64 bytes from arn09s22-in-f4.1e100.net (142.250.74.36): icmp_seq=4 ttl=53 time=8.15 ms

--- www.google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3007ms
rtt min/avg/max/mdev = 6.888/7.889/9.569/1.092 ms
```

# Github helper scripts

In [this repository](https://github.com/straxy/qemu-board-emulation) I have added several scripts which cover most of the things presented in Parts 1 and 2.

Following scripts are present:

* `install-qemu.bash` - downloads toolchain and Ubuntu root filesystem, and compiles QEMU, U-Boot and Linux
* `prepare-qemu.bash` - creates an SD card image based on compiled files and Ubuntu root filesystem
* `enable-networking.bash` - initializes a tap network interface so QEMU instance can have networking
* `mount-sd-card.bash` - mounts rootfs partition of the SD card
* `umount-sd-card.bash` - umounts rootfs partition of the SD card
* `run-qemu.bash` - runs QEMU instance

# Summary

In this post we have covered different methods for running the complete Linux system with bootloader in QEMU. For now, we have only used Versatile Express V2P-CA9 board.

In the following posts we will explore different functionalities that are emulated by QEMU for the Versatile express board~~, and also look at NXP iMX6 SabreLite and OrangePi PC boards~~.
