# rtw88 backport for the SC2 board's RTL8822CS

The board's SDIO WiFi module is a **Realtek RTL8822CS** (`024c:c822`). The
vendor 5.4 kernel's in-tree `rtw88` driver supports only PCIe 8822BE/CE,
so the SDIO driver comes from the lwfinger downstream backport:

    https://github.com/lwfinger/rtw88
    commit f1dd7c3  (the base this patch applies to)

`0001-sc2-5.4-compat.patch` contains the changes needed to build it
against the vendor 5.4 tree:

- the Makefile is trimmed to the four modules the 8822CS needs
  (`rtw_core`, `rtw_sdio`, `rtw_8822c`, `rtw_8822cs`), so unrelated
  chipsets cannot trip over vendor-5.4 API gaps;
- `device_get_mac_address()` is called with the 5.4 three-argument form;
- the `guard()` cleanup helpers (not in 5.4) are replaced with explicit
  lock/unlock.

`SC2-NOTES.md` records the bring-up, the module ABI trap and the WPA2
association test. The modules that ship in the image are built from this
source against the same kernel as `kernel/Image`, then stripped of debug
information. The build recipe is in [`docs/build-the-kernel.md`](../../docs/build-the-kernel.md#build-the-wifi-modules).

The driver is kernel code: `Dual BSD/GPL` per its module metadata. The
firmware (`rtw8822c_fw.bin`, `rtw8822c_wow_fw.bin`) is already in the
Debian image under `/lib/firmware/rtw88/`, installed by the distribution's
firmware package; it is not rebuilt here.
