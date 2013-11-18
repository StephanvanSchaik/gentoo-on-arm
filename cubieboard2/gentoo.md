# Setting up a cross-compiler.

If you don't have a host computer with the same architecture as the target you
are going to be compiling for (armv7a-hardfloat), you'll have to set up a
cross-compiler for the target architecture, if you haven't already.

Let's install crossdev first, which allows us to easily build the toolchain:

```
emerge -av crossdev
```

Then we can build a stable tool chain for the target
(armv7-hardfloat-linux-gnueabi):

```
crossdev -S -t armv7-hardfloat-linux-gnueabi
```

Grab a cup of coffee, as this may take a while.

# Building u-boot and the SPL.

Now that we have a working toolchain, we can start building das u-boot and the
accompanying SPL. Das u-boot SPL is open-source firmware that gets loaded when
the device is being powered on, and it is mostly responsible for kickstarting
u-boot, which on its own is responsible for setting up a proper environment for
the kernel and loading the kernel.

The Cubieboard 2 has its very own fork for u-boot, let's clone it:

```
git clone https://github.com/linux-sunxi/u-boot-sunxi.git
```

Now that the fork has been cloned, we can build the binaries that we need to
boot the system:

```
cd u-boot-sunxi
CROSS_COMPILE=arm-hardfloat-linux-gnueabi- make distclean
CROSS_COMPILE=arm-hardfloat-linux-gnueabi- make Cubieboard2_config
CROSS_COMPILE=arm-hardfloat-linux-gnueabi- make
```

If you are interested in booting via a USB-device rather than a SD-card, then
you can use Cubieboard2_FEL_config instead of Cubieboard2_config. If you are
interested in booting from NAND Flash, then you'll have to switch to the
lichee-dev-a20 branch. More possible configurations can be found in
u-boot-sunxi/boards.cfg.

If everything went all right, you should now have two binary files that we will
be using:

 - u-boot-sunxi/u-boot.bin
 - u-boot-sunxi/spl/u-boot-spl.bin

TODO: is using mainline u-boot feasible?

# Formatting.

Let's format the USB storage device/SD-card. We can do this is by using an
utility such as fdisk. A sane default is to set up the partitions like this:

 - boot (mark as bootable):
   - start: 2M (default).
   - size: 64M
 - root:
   - start: 66M (default).
   - size: whatever is left on the device.

You can run the fdisk utility on the SD-card using:

```
fdisk /dev/mmcblk${n}
```

Where ${n} is the number of your SD-card, or, alternatively, you can run the
fdisk utility on your USB storage device:

```
fdisk /dev/sd${x}
```

Where ${x} is the letter of your USB storage device.

After that you can format the boot and root partitions to whatever file system
formats you like, as long as u-boot and Linux support those respectively. In our
case we will choose for ext2 and ext4 as the default:

```
mkfs.ext2 /dev/mmcblk${n}p1
mkfs.ext4 /dev/mmcblk${n}p2
```

Or for USB:

```
mkfs.ext2 /dev/sd${x}1
mkfs.ext4 /dev/sd${x}2
```

We will be refering to these devices with ${disk}, ${bootfs} and ${rootfs}
respectively.

# Installing u-boot.

The region with offset 8K and size 32K contains the SPL initial loader, so let's
write that away to the disk:

```
dd if=u-boot-sunxi/spl/u-boot-spl.bin of=${disk} bs=1024 seek=8
```

The next region of 512K contains u-boot:

```
dd if=u-boot-sunxi/u-boot.bin of=${disk} bs=1024 seek=40
```

Finally, the next 128K contains the environment, let's clear that:

```
dd if=/dev/zero of=${disk} bs=1024 count=128 seek=552
```

# Setting up the root partition.



# Building the Linux kernel.


