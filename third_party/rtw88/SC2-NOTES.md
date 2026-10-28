# rtw88 backport for the SC2 board's RTL8822CS (024c:c822)

This directory is a clone of <https://github.com/lwfinger/rtw88>, the
downstream backport of the mainline rtw88 driver. It is here because the
board's SDIO WiFi module is a **Realtek RTL8822CS**, not a Broadcom part
(the DTB's `brcm,bcm4329-fmac` compatible string is misleading vendor
glue).

The vendor 5.4 kernel tree only has PCIe rtw88 (8822BE/CE), no SDIO. The
lwfinger tree supports 8822CS over SDIO and claims kernel 5.4+.

## Local changes

- `Makefile` trimmed to the four modules needed for 8822CS:
  `rtw_core`, `rtw_sdio`, `rtw_8822c`, `rtw_8822cs`. The original is
  `Makefile.upstream`. The full Makefile builds every chipset and trips
  over unrelated 5.4 API gaps (`FIELD_PREP_CONST`, etc.).
- `main.c`: include `<linux/property.h>` and use the 5.4 three-argument
  `device_get_mac_address()`.
- `sdio.c`: replace the `guard()` cleanup helpers (not present in 5.4)
  with explicit `mutex_lock`/`mutex_unlock` and
  `spin_lock_irqsave`/`spin_unlock_irqrestore`.

## Build

**Build against a config that matches the running kernel**, plus
`CONFIG_MAC80211=m` and `CONFIG_CRYPTO_CCM=m`. A mismatched config is not
harmless: the first attempt used `build/cfgWifi`, which differed from the
running kernel in 118 options, and loading the modules hard-hung the box.

The working output directory is `build/cfgWifiExact`, a copy of the
running kernel's `/proc/config.gz` with only those two options added. It
differs from the running config by exactly those lines:

    export ARCH=arm64
    export CROSS_COMPILE=<toolchain>-
    make -C build/linux O=$PWD/build/cfgWifiExact \
         M=$PWD/third_party/rtw88 -j$(nproc) modules

The modules carry vermagic `5.4.210-ga3aaaa5b0e64-dirty ... modversions`.

## Test status — works

The rebuilt modules load and the RTL8822CS probes:

    rtw_8822cs mmc1:0001:1: Firmware version 9.9.6, H2C version 15
    rtw_8822cs mmc1:0001:1: WOW Firmware version 9.9.4, H2C version 15

`wlan0` appears (MAC `ac:f4:2c:a9:d8:47`), and `iw dev wlan0 scan`
returns BSS entries without wedging the kernel. A cold boot autoloads the
stack via `/etc/modules-load.d/rtw88.conf`.

Association works. The first attempt reached the 4-way handshake but
failed with `Failed to set PTK to the driver`: the running kernel has no
CCM crypto module, so mac80211 could not install the WPA2 key. Build and
install `crypto/ccm.ko` as well:

    make -C build/linux O=$PWD/build/cfgWifiExact \
         CRYPTO_CFLAGS="-I$PWD/build/sslprefix/usr/include" \
         CRYPTO_LIBS="-L$PWD/build/sslprefix/lib -lcrypto" \
         crypto/ccm.ko

Copy `crypto/ccm.ko` to
`/lib/modules/5.4.210-ga3aaaa5b0e64-dirty/extra/` and add `ccm` to
`/etc/modules-load.d/rtw88.conf`. With CCM loaded the box associates to
`TP-Link_8860`, gets DHCP, and passes traffic. A cold boot reconnects
automatically through `wpa_supplicant@wlan0.service` (the generic
`wpa_supplicant.service` is disabled to avoid contention).

The five rtw88 modules installed in
`/lib/modules/5.4.210-ga3aaaa5b0e64-dirty/extra/` hash-match the `.ko`
files in this directory. `mac80211.ko` is not built here; it comes from
the kernel build with `CONFIG_MAC80211=m`.
