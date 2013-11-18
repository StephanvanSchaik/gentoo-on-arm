TODO: add reference to page explaining how to install the overlay.

# Setting up the kernel modules.

The first step in getting video decoding to work is to load the module to
actually access the VPU. The module that is used for this by the sunxi/linux-3.4
kernel is sunxi_cedar_mod. You can load this by issuing the following command:

```
modprobe sunxi_cedar_mod
```

If you always want the module to be loaded, then add the following line to
/etc/conf.d/modules:

```
modules="sunxi_cedar_mod"
```

Upon loading the module, /dev/cedar_dev will be present, which will be used by
the driver in addition to /dev/disp. In order to make these usable for the video
user group, we will add a set of rules to udev. Add the following rules to a new
file called /etc/udev/rules.d/90-sunxi-video.rules:

```
KERNEL=="disp", MODE="0770", GROUP="video"
KERNEL=="cedar_dev", MODE="0770", GROUP="video"
```

# Installing VDPAU.

There is currently an experimental VDPAU driver available for the Cubieboard
that is capable of decoding MPEG-1, MPEG-2 and H.264 using the VPU hardware.
Before we can actually install the driver, we have to install the VDPAU library
itself, as well as a useful tool for checking if VDPAU actually works. However,
since VDPAU isn't really being used on ARM at the very moment, the packages are
blocked, you can unblock them by adding the following lines to
/etc/portage/package.keywords:

```
x11-libs/vdpau **
x11-misc/vdpauinfo **
```

Whereupon you can install them by running:

```
emerge -av libvdpau vdpauinfo
```

# Installing the driver.

After you have installed the library, you can install the driver itself. At the
very moment the driver is available at
https://github.com/linux-sunxi/libvdpau-sunxi, but since we are using the
overlay we can simply install the driver by unblocking the package by adding the
following to /etc/portage/package.keywords:

```
x11-libs/libvdpau-sunxi ~arm
```

And by running:

```
emerge -av libvdpau-sunxi
```

Now that the driver is installed, you have to make your software aware of its
presence, since most, if not all, VDPAU-enabled software will default to the
Nvidia-implementation. You can do this by adding the following to
/etc/env.d/99-vdpau:

```
VDPAU_DRIVER=sunxi
```

# Installing a VDPAU-enabled media player.

Before we can actually start using VDPAU, we have to install a VDPAU-enabled
media player such as mpalyer. Even though the guide will only cover installing
and using mplayer, you are free to use whatever you like, as long as it offers
support for VDPAU.

You can enable vdpau for mplayer by adding it to your USE-flags, or by adding
the following to /etc/portage/package.use:

```
media-video/mplayer vdpau
media-video/ffmpeg vdpau
```

However, since VDPAU is still considered unstable on ARM, you'll have to add the
following lines to /etc/portage/profile/package.use.mask before the USE-flags
will have any effect:

```
media-video/mplayer -vdpau
media-video/ffmpeg -vdpau
```

Then you can install mplayer:

```
emerge -av mplayer
```

If you want to use mplayer2 instead of mplayer, just replace mplayer with
mplayer2. The instructions are the same.

# Testing the video decoding.

First of all, you should check if VDPAU itself is working by running vdpauinfo.
If everything is fine, it will tell you what codecs are being supported by your
driver.

Once you have confirmed that it is all right, you can download and try playing
any of the following films:

 - http://www.sintel.org/download
 - http://www.bigbuckbunny.org/index.php/download/

You should then be able to play the H.264-encoded files, at the least, by
issuing the following command:

```
mplayer -vo vdpau -vc ffmpeg12vdpau,ffh264vdpau [path]
```

