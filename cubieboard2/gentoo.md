# Setting up a cross-compiler.

If you don't have a host computer with the same architecture as the target you
are going to be compiling for (armv7a-hardfloat), you'll have to set up a
cross-compiler for the target architecture, if you haven't already.

Let's install crossdev first, which allows us to easily build the toolchain:

```
emerge -av crossdev
```

Then we can build a stable tool chain for the target
(armv7a-hardfloat-linux-gnueabi):

```
crossdev -S -t armv7a-hardfloat-linux-gnueabi
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
git clone --depth 1 https://github.com/linux-sunxi/u-boot-sunxi.git
```

Now that the fork has been cloned, we can build the binaries that we need to
boot the system:

```
cd u-boot-sunxi
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- make distclean
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- make Cubieboard2_config
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- make
```

If you are interested in booting via a USB-device rather than a SD-card, then
you can use Cubieboard2_FEL_config instead of Cubieboard2_config. If you are
interested in booting from NAND Flash, then you'll have to switch to the
lichee-dev-a20 branch. More possible configurations can be found in
u-boot-sunxi/boards.cfg.

If everything went all right, you should now have two binary files that we will
be using:

 - u-boot-sunxi/u-boot.img
 - u-boot-sunxi/spl/sunxi-spl.bin

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
dd if=u-boot-sunxi/spl/sunxi-spl.bin of=${disk} bs=1024 seek=8
```

The next region of 512K contains u-boot:

```
dd if=u-boot-sunxi/u-boot.img of=${disk} bs=1024 seek=40
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
${bootfs}       /boot                   ext2    noauto,noatime          1 2
${rootfs}       /                       ext4    noatime                 0 1
none            /var/tmp/portage        tmpfs   size=1024M,noatime      0 0
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

For everything else refer to the [Networking Information](http://www.gentoo.org/doc/en/handbook/handbook-arm.xml?part=1&chap=8#doc_chap2)
section in the Gentoo handbook for ARM.

# Building the Linux kernel.

Since Allwinner A20 isn't being fully supported by the mainline Linux kernel
yet, we'll be cloning the sunxi-linux repository, or more specifically, the
latest revision of the sunxi-3.4 branch:

```
git clone --depth 1 git://github.com/linux-sunxi/linux-sunxi.git -b sunxi-3.4
cd linux-sunxi
```

After cloning the repository, we have to configure the kernel for the board in
question. In addition to the default configuration that is available within the
cloned repository, there is also a [stripped-down version](
../configs/sun7i-config), for which all optional modules have been disabled:

```
wget https://github.com/StephanvanSchaik/gentoo-on-arm/configs/sun7i-config -O .config
```

Or, if you'd rather want to use the default configuration:

```
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- ARCH=arm make sun7i_defconfig
```

Whereupon we can start building the kernel and its modules (for a faster
compilation, you can specify -jn, where n is the amount of threads + 1):

```
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- ARCH=arm make -j3 uImage modules
```

# Installing the kernel.

Installing the kernel and its modules is pretty straight-forward:

```
mount ${bootfs} /mnt/gentoo/boot
cp arch/arm/boot/uImage /mnt/gentoo/boot/
CROSS_COMPILE=armv7a-hardfloat-linux-gnueabi- ARCH=arm make INSTALL_MOD_PATH=/mnt/gentoo modules_install
```

# Further configuration.

The kernel also requires a configuration file that tells a few things about the
hardware on the board, but it has to be in a specific format for the kernel in
order for it to be able to use it. Fortunately, there is a tool named fex2bin
that will convert it to the appropriate format:

```
git clone --depth 1 git://github.com/linux-sunxi/sunxi-tools.git
cd sunxi-tools
make fex2bin
```

After fex2bin has been compiled, we can clone the repository that contains the
various board configurations:

```
git clone --depth 1 git://github.com/linux-sunxi/sunxi-boards.git
```

In our case we are interested in sunxi-boards/sys_config/a20/cubieboard2.fex,
feed it to fex2bin, so that we can get the script.bin file, and store it on the
boot partition:

```
fex2bin cubieboard2.fex /mnt/gentoo/boot/script.bin
```

# Finalising.

The last step is writing the boot script that loads the hardware configuration
and boots the kernel. The following boot script will do that, as well as setting
up both a serial console as a framebuffer console. The resolution to be set for
the framebuffer will be determined by using EDID, and if that fails, it will
fallback to the resolution specified in the boot script. Edit
/mnt/gentoo/boot/boot.cmd:

```
setenv bootargs console=tty0 hdmi.audio=EDID:0 disp.screen0_output_mode=EDID:1920x1080p60 console=ttyS0,115200 root=${rootfs} rootwait panic=10 ${extra}
ext2load mmc 0 0x43000000 script.bin
ext2load mmc 0 0x48000000 uImage
bootm 0x48000000
```

And make an image of it:
```
cd /mnt/gentoo/boot/
mkimage -C none -A arm -T script -d boot.cmd boot.scr
```

Just unmount the two mounted partitions:

```
umount /mnt/gentoo/boot
umount /mnt/gentoo/
```

