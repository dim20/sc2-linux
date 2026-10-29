# Hardware and boot chain

The board is an **EON SDSTB02** (Amlogic S905X4 / SC2) with 2 GB LPDDR4
and 14.6 GB eMMC. The vendor bootloader is locked; this project boots a
custom 5.4 kernel from the USB stick or the eMMC.

## Other S905X4 boxes

**You may not need any of this.**

A lot of the difficulty here is specific to *this* board and *this*
bootloader:

- the vendor bootloader cannot `booti` a PE/ZBOOT-wrapped kernel, which is
  what essentially every distro ships — so a custom kernel has to be built;
- the internal Ethernet PHY lives on a private MDIO bus that mainline's
  device tree does not describe, so mainline `stmmac` cannot drive it;
- the device tree shipped with the vendor kernel has two variants and the
  wrong one aborts before the kernel prints anything.

If your box already boots mainline Linux from a USB stick, you need none
of this and should stop reading. If you are hitting one of those specific
walls, this repo may save you a week.

The kernel is built from **CoreELEC's `amlogic-5.4.210`** tree, which
already supports this SoC properly. That is the single biggest reason
this project is tractable.

## What the box does on power cycle

On power cycle, with nothing connected:

```
BL2 → BL31 → U-Boot
  "Hit any key to stop autoboot:  3  2  1  0"     ← 3 s to reach the U-Boot prompt
  bootcmd (saved in the U-Boot environment, set by the installer)
    usb start 0
    if fatload usb 0:1 0x2000000 Image; then
        → boot from the USB stick, root on /dev/sda2
    else
        mmc dev 1; mmc rescan
        ext4load mmc 1:1 0x2000000 Image
        ext4load mmc 1:1 0x5000000 sc2_s905x4_ah212_linux.dtb
        setenv bootargs 'root=/dev/mmcblk0p2 ... init=/lib/systemd/systemd'
        booti 0x2000000 - 0x5000000
        → Linux 5.4.210, root on eMMC, systemd as PID 1
```

The stick is a **recovery key**, not a dependency: plug it in and the box
boots the stick; unplug it and the box boots the eMMC. No menu. The
three-second U-Boot window exists only when no stick is attached: the
vendor's own pre-boot chain runs the stick's autoscript before the
countdown.

**The serial port is the kernel console; the screen is a virtual
terminal.** The kernel command line keeps `console=ttyS0,921600` and
nothing else, so the full boot log goes to the serial port and stays
there, and `getty` on `tty1` gives you a login prompt on HDMI — the usual
Linux arrangement, with `Ctrl-Alt-F1`…`F6` for the other terminals.

Adding `console=tty0` would put the kernel's own messages on the screen as
well, and it was tried. It is not what puts the login prompt there (that
is `getty` + the framebuffer console), and it means a driver logging a
line at 62 seconds lands in the middle of the login prompt. Left out on
purpose.

## Recovery

The recovery ladder — UART, USB stick, kernel rollback, DNL/fastboot — is
in [`recovery.md`](recovery.md). The short version: the stick is a complete
recovery system, and the U-Boot prompt is available for three seconds when
no stick is attached.
