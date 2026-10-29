# Limitations and known gaps

These are real and will bite you. This is the detailed list; the landing
page keeps only the short version.

1. **Audio over HDMI works, but only with one mixer control set.** The
   transmitter's `HDMITX Audio Source Select` defaults to `Spdif`, while the
   playback path is `Tdm_C` (`hw:0,2`); with the default the stream runs and
   nothing is heard. Set it to `Tdm_C` — the image now does this at boot —
   and a 440 Hz test tone is audible on the TV. Analog (T9015) and SPDIF
   outputs were **not** tested. See [`audio.md`](audio.md).
2. **There is no desktop on the TTY image.** That is deliberate: a
   display manager takes the framebuffer from the text console, and a
   desktop is a large amount of software the user may not want.

   `v0.6-desktop` did ship one, and its terminal text was unreadable —
   the X server's screen pixmap was 24 bits deep inside a 32-bit
   framebuffer. The fix needs upstream's `fb24_32` conversion, which this
   version of the fbdev driver does not have. The desktop is frozen; the
   plan is an optional package set in a later release. If you are running
   v0.6 and want the console, mask the display manager:

   ```sh
   systemctl disable --now lightdm
   systemctl mask lightdm
   reboot
   ```

   Do **not** expect `tasksel` to give you a working desktop yet: it will
   install one, and the text will be wrong for the reason above.
3. **No GPU-accelerated 3D.** The vendor DRM driver advertises no dma-buf
   modifier support, so Mesa's GBM has nothing to negotiate. A 2D desktop
   does not need it; 3D does.
4. **The clock fallback is only unit-tested.** The chrony success path is
   proven on hardware; the "no network at boot" path has not been
   exercised on a real box.
5. **The root password is published.** `root`/`tvbox` is in a public
   repository. Change it before the box is on a network you do not trust:

   ```sh
   passwd
   ```

   The box also accepts root logins over SSH with that password. There is
   no first-boot wizard yet; this is documented rather than fixed.
6. **The kernel has no upstream security updates.** It is a vendor tree,
   Linux 5.4.210, built from CoreELEC/Amlogic sources. The kernel package
   makes a new kernel installable and a bad one reversible, but it does
   not track upstream.
7. **Only one box has been tested.** Everything here was proven on one
   development box. It has not been validated on other hardware.
8. **The first boot may need the vendor's own update path.** If your box
   does not find `aml_autoscript` on the stick by itself, see
   [`recovery.md`](recovery.md) — you will need a UART console
   once.
9. **Opening `/dev/fb0` panics the box — on v0.5.1, v0.6, and any image
   built before v0.7.** v0.7-tty fixes it. The text console itself is
   fine, and it draws all day. But any program that opens `/dev/fb0` —
   `fbset`, a framebuffer tool, an X server — closes it again, and on
   those older images the kernel panics a few seconds later: the
   framebuffer console's cursor puts the mapping back down and the next
   blink writes through a NULL screen base.

   ```
   fb_flashcursor -> bit_cursor -> soft_cursor ->
   drm_fb_helper_cfb_imageblit -> cfb_imageblit   (write to NULL + 0x200)
   Kernel panic - not syncing: Fatal exception
   ```

   Reproduced on 2026-10-23 on the test box, with the serial console
   capturing the panic. **On an image older than v0.7-tty, do not open
   `/dev/fb0`.** The fix is in the v0.7 kernel; see
   [`framebuffer.md`](framebuffer.md) for the mechanism.

If you want to help finish it, see the [roadmap](development.md#roadmap).
