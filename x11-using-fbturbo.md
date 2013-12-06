# Introduction

fbturbo, formerly sunxifb, is an improved fork of the fbdev DDX driver that
offers features such as 2D hardware acceleration on platforms where it is
available. This DDX driver is recommended for various ARM SoCs, such as the
Samsung Exynos SoCs and the Allwinner SoCs.

# Installing fbturbo.

Before we can install fbturbo and its dependencies using the overlay, we have to
unblock the packages. Add the following lines to /etc/portage/package.keywords:

```
x11-libs/libdri2 ~arm
x11-drivers/xf86-video-fbturbo ~arm
```

Then we can install the package by running:

```
emerge -av xf86-video-fbturbo
```

# Configuring X.

We now have to configure X to use fbturbo, for this we will be editing
/etc/X11/xorg.conf.d/10-fbturbo.conf. In our example we'll assume that we have
two screens attached to our Cubietruck, one via a VGA-connector and the other
via a HDMI-connector. Both screens have a native resolution of 1920x1080, and
they are set up in such a way that they are to be used side by side. Let's start
by adding a ServerLayout section that describes what screens there are, and
where they are located:

```
Section "ServerLayout"
	Identifier	"Default Layout"
	Screen		0 "VGA-0" 0 0
	Screen		0 "HDMI-0" 1920 0
EndSection
```

After that, we want to specify what devices the screens are attached to. In case
of the Allwinner SocS, the driver will map each screen to its own framebuffer,
up to two framebuffers. In the case of other SoCs, the screens may be mapped to
the same framebuffer. Since the example deals with a Cubietruck, the screen
connected via VGA will be exposed as /dev/fb0, whereas the one connected via
HDMI will be exposed as HDMI. To properly identify the screens, we'll be using
the connector type, as well as a number for the identifier.

In addition, we'll add an option to turn off DPMS for the screens. The reason
why this is being done, is because some issues may arise when using this with
the fbturbo driver. If this is not the case for you, then you may leave out that
option, since DPMS is enabled by default.

```
Section "Screen"
	Identifier	"VGA-0"
	Device		"/dev/fb0"
	Monitor		"LG"
	Option		"DPMS" "false"
EndSection

Section "Screen"
	Identifier	"HDMI-0"
	Device		"/dev/fb1"
	Monitor		"LG"
	Option		"DPMS" "false"
EndSection
```

Now we have to set up the devices for every framebuffer that we want to use.
Since our example relies on the Cubietruck that is being configured for
dual-head display, we'll be having two devices, one for /dev/fb0 and another for
/dev/fb1:

```
Section "Device"
	Identifier	"/dev/fb0"
	Driver		"fbturbo"
	Option		"fbdev" "/dev/fb0"
	Option		"SwapBuffersWait" "true"
EndSection

Section "Device"
	Identifier	"/dev/fb1"
	Driver		"fbturbo"
	Option		"fbdev" "/dev/fb1"
	Option		"SwapBuffersWait" "true"
EndSection
```

Furthermore, you may want to configure the DPMS timings:

```
Section "ServerFlags"
	Option		"BlankTime" "0"
	Option		"StandbyTime" "0"
	Option		"SuspendTime" "0"
	Option		"OffTime" "0"
EndSection
```

