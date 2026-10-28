# Loadable modules for the shipped 5.4 kernel

The kernel Image is monolithic except for the WiFi stack. Two in-tree
modules and four out-of-tree modules are loadable:

| Module | Source | Why it is a module |
|---|---|---|
| `mac80211.ko` | the vendor 5.4 tree, `CONFIG_MAC80211=m` | the rtw88 backport links against its exported symbols |
| `ccm.ko` | the vendor 5.4 tree, `CONFIG_CRYPTO_CCM=m` | mac80211 needs CCM for WPA2; the vendor defconfig omits it |
| `rtw_core.ko`, `rtw_sdio.ko`, `rtw_8822c.ko`, `rtw_8822cs.ko` | the lwfinger rtw88 backport | the vendor tree's rtw88 driver has no SDIO support |

The modules are copied into the image at
`/lib/modules/5.4.210-ga3aaaa5b0e64-dirty/extra/` and loaded by
`/etc/modules-load.d/rtw88.conf` at boot. They must be built against the
same kernel as `kernel/Image`; `modprobe` refuses a module whose vermagic
does not match. Update the Image and this directory together.

## Provenance

The in-tree modules come from the same vendor tree and config as the
Image. The four rtw88 modules are built from the lwfinger backport at
<https://github.com/lwfinger/rtw88>, trimmed to the 8822CS chipset. The
local compatibility patch and build notes are in
`third_party/rtw88/`; the public source is GPL-2.0 and the modules carry
`Dual BSD/GPL` metadata.

The modules are stripped of debug information (`strip --strip-debug`) so
the image does not carry 36 MB of DWARF. The code and ABI are unchanged;
a cold boot with the stripped set was tested on the board.

## Checksums

`SHA256SUMS` covers the module files in the version directory. Verify a
downloaded or rebuilt set with:

    cd kernel/modules/5.4.210-ga3aaaa5b0e64-dirty
    sha256sum -c ../SHA256SUMS
