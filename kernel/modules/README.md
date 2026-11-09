# Loadable modules for the shipped 5.4 kernel

The Image is built with Docker/containerd/runc support. Most of the kernel
is still monolithic, but the WiFi stack and the optional filesystems that
do not need to be present at boot are loadable:

| Modules | Source | Why they are modules |
|---|---|---|
| `mac80211.ko` | the vendor 5.4 tree, `CONFIG_MAC80211=m` | the rtw88 backport links against its exported symbols |
| `rtw_core.ko`, `rtw_sdio.ko`, `rtw_8822c.ko`, `rtw_8822cs.ko` | the lwfinger rtw88 backport | the vendor tree's rtw88 driver has no SDIO support |
| `btrfs.ko`, `xfs.ko`, `cifs.ko`, `nfs*.ko` | the vendor 5.4 tree | large, optional at boot; built in they overflow the 48 MiB BOOT partition |
| `dm-thin-pool.ko`, `dm-persistent-data.ko`, `dm-bio-prison.ko`, `nbd.ko` | the vendor 5.4 tree | optional storage targets |
| `zstd_*.ko`, `xxhash.ko`, `xor*.ko`, `raid6_pq.ko` | the vendor 5.4 tree | dependencies of btrfs |

`ccm` is **built in** (`CONFIG_CRYPTO_CCM=y`), not a module: CIFS selects it
and mac80211 uses it through the crypto API. `f2fs` is also built in: the
vendor tree defines `trace_android_fs_*` in ext4 without `EXPORT_SYMBOL`,
so an `f2fs.ko` cannot resolve those symbols and refuses to load.

The modules are copied into the image at
`/lib/modules/5.4.210-ga3aaaa5b0e64-dirty/extra/` and loaded on demand by
`modprobe`; the WiFi stack is loaded at boot by
`/etc/modules-load.d/rtw88.conf`. They must be built against the same
kernel as `kernel/Image`; `modprobe` refuses a module whose vermagic or
symbol CRCs do not match. Update the Image and this directory together.

## Provenance

The in-tree modules come from the same vendor tree and config as the
Image. The four rtw88 modules are built from the lwfinger backport at
<https://github.com/lwfinger/rtw88>, trimmed to the 8822CS chipset. The
local compatibility patch and build notes are in
`third_party/rtw88/`; the public source is GPL-2.0 and the modules carry
`Dual BSD/GPL` metadata.

The modules are stripped of debug information (`strip --strip-debug`) so
the image does not carry tens of megabytes of DWARF. The code and ABI are
unchanged; a cold boot with the stripped set was tested on the board.

## Checksums

`SHA256SUMS` covers the module files in the version directory. Verify a
downloaded or rebuilt set with:

    cd kernel/modules/5.4.210-ga3aaaa5b0e64-dirty
    sha256sum -c ../SHA256SUMS
