# What does work?

 - MicroSD.
 - USB.
 - Framebuffer console (HDMI output with support for EDID).
 - X11 using fbturbo.
 - Hardware-accelerated video decoding using the open-source sunxi-vdpau
   implementation for MPEG-1, MPEG-2 and H.264.
 - Ethernet.
 - Audio using the on-board audio chipset.

# What doesn't work?

 - The LEDs aren't controllable via sunxi/linux-3.4.
 - There is an issue where the monitor blanks for a second, a few minutes upon
   booting.
 - DPMS, for some monitors upon turning off, they can't be turned on again
   using DPMS.

# What hasn't been tested?

 - Anything that uses proprietary drivers (such as video decoding and OpenGL
   ES).
 - SATA (TODO: get the HDD add-on).
 - HDMI audio.
 - Mali in general (TODO: wait for Lima).

# Upstream progress

If you are, just like me, interested in the upstream progress, so that we can
eventually get to use gentoo-sources instead of the fork, then you should keep
your eye on http://linux-sunxi.org/Linux_mainlining_effort

