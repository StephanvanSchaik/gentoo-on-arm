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

Choose your [http://www.gentoo.org/main/en/mirrors2.xml](favourite mirror) to
download the armv7a-hardfloat-linux-gnueabi stage3 tarball from, and extract it
to your root partition upon mounting it:

```
mkdir -p /mnt/gentoo/
mount ${rootfs} /mnt/gentoo/
tar xjpf stage3-armv7a_hardfp-*.tar.bz2 -C /mnt/gentoo
```

## Setting up fstab.

Edit the /etc/fstab to look this:

```
${bootfs}	/boot			ext2
${rootfs}	/			ext4
none		/var/tmp/portage	tmpfs	size=1024M,noatime	0 0
```

Do notice that the path where Portage stores its temporary files is set up in
such a fashion that it will use tmpfs (i.e. that it will use the board's
memory). This set-up is recommended as it will lengthen the durability of your
storage device, and as it will speed up the compilation time with a significant
amount.

However, some packages, such as Firefox and gcc, may use more of this space than
there is memory available. For these we'll use a configuration that tells
Portage to use another location (this may be on the rootfs, or a symbolic link
to another, faster device), create /etc/portage/env/notmpfs.conf with the
following contents:

```
PORTAGE_TMPDIR="/var/tmp/notmpfs"
```

Packages that should use this configuration can then be added to
/etc/portage/package.env, for instance:

```
sys-devel/gcc notmpfs.conf
www-client/firefox notmpfs.conf
```

## Configuring the network settings.

The ethernet driver may not be loaded automatically, in that case you can
modprobe it:

```
modprobe sunxi_emac
```

Or you can add the module to /etc/conf.d/modules, if you want it to be always
loaded on boot:

```
modules="... sunxi_emac"
```

Another thing to keep in mind is that the device does not have an EEPROM with
information on the network interface controller such as the MAC-address.
Therefore a random MAC-address is generated upon boot, however, if you desire to
use a static MAC-address, you can set one in /etc/conf.d/net:

```
mac_eth0="00:00:00:00:00:00"
```

For everything else refer to the
[http://www.gentoo.org/doc/en/handbook/handbook-arm.xml?part=1&chap=8#doc_chap2]
(Networking Information) section in the Gentoo handbook for ARM.

# Building the Linux kernel.

Since the Cubieboard 2 isn't being fully supported by the mainline Linux kernel
yet, we'll be cloning the sunxi-linux repository:

```
git clone git://github.com/linux-sunxi/linux-sunxi.git
```

Now that we have cloned the kernel, we have to check out the right branch, for
the Cubieboard 2 the sunxi-3.4 branch is fine. In addition to that we'll need a
good default configuration for the kernel, since the Cubieboard 2 uses an
Allwinner A20, we'll be using sun7i_defconfig:

```
cd linux-sunxi
git checkout --track origin/sunxi-3.4
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- ARCH=arm make sun7i_defconfig
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- ARCH=arm make uImage modules
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- ARCH=arm make
INSTALL_MOD_PATH=${rootfs} modules_install
```

