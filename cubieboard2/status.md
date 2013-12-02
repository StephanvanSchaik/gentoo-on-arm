# What does work?

 - MicroSD.
 - SATA (tested with a 3.5" HDD, you have to be careful with the current though).
 - USB.
 - /dev/fb0 using HDMI.
 - X11 using fbturbo.
 - Hardware-accelerated video decoding using the open-source sunxi-vdpau
   implementation for MPEG-1, MPEG-2 and H.264.
 - 10/100 ethernet.
 - Audio using both the chipset and HDMI audio.
 - LEDs.

# What doesn't work?

 - There may be some issues with the sunxi-video driver, mostly because the
   implementation is cumbersome.

# What hasn't been tested?

 - Anything that uses proprietary drivers (such as video decoding and OpenGL
   ES).
 - Mali (I will do this once there is an official release of lima).
 - USB gadgets (they may work, I still have to try it).
 - Anything else that hasn't been listed here (such as GPIO).

# Upstream progress

If you are, just like me, interested in the upstream progress, so that we can
eventually get to use gentoo-sources instead of the fork, then you should keep
your eye on http://linux-sunxi.org/Linux_mainlining_effort

