# What does work?

 - MicroSD.
 - USB.
 - /dev/fb0 using HDMI with support for EDID parsing.
 - X11 using fbturbo.
 - Hardware-accelerated video decoding using the open-source sunxi-vdpau
   implementation for MPEG-1, MPEG-2 and H.264.
 - 10/100 ethernet.
 - Audio using both the chipset and HDMI audio.

# What doesn't work?

 - The LEDs aren't controllable in sunxi/linux-3.4 (I have figured this
   one out; I will test this sometime soon).
 - There is an issue where the monitor blanks for a second, a few minutes upon
   booting.
 - For some monitors DPMS may not work properly. If that is the case with your
   monitor, then try turning off DPMS.

# What hasn't been tested?

 - Anything that uses proprietary drivers (such as video decoding and OpenGL
   ES).
 - SATA (I have *ordered* the hardware necessary for this).
 - Mali (I will do this once there is an official release of lima).
 - USB gadgets (they may work, I still have to try it).
 - Anything else that hasn't been listed here (such as GPIO).

# Upstream progress

If you are, just like me, interested in the upstream progress, so that we can
eventually get to use gentoo-sources instead of the fork, then you should keep
your eye on http://linux-sunxi.org/Linux_mainlining_effort

