# WiFi overrides

The image ships the Realtek RTL8822CS SDIO stack as loadable modules. The
modules themselves come from `kernel/modules/`; this directory carries the
userspace files that make them useful:

- `modules-load.d/rtw88.conf` loads `ccm` and `rtw_8822cs` at boot.
- `wpa_supplicant/wpa_supplicant-wlan0.conf` is a credential-free template,
  installed as `/root/wpa_supplicant.conf` (the vendor's
  `/etc/wpa_supplicant/wpa_supplicant-wlan0.conf` symlinks to it).

The build enables `wpa_supplicant@wlan0.service` and masks the generic
`wpa_supplicant.service`, so two supplicants do not contend for `wlan0`.
