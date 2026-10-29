# Network

What the box does out of the box, and the recipes for the three things
people ask for: a static address, mDNS, and WiFi. The Ethernet and mDNS
sections were read from the eMMC system on **2026-10-23**; the WiFi
section was updated on **2026-10-28** from the development box.

---

## The default: DHCP on Ethernet

`systemd-networkd` is the network manager. It is enabled, and so is
`systemd-resolved`.

`/etc/systemd/network/10-eth0.network`:

```
[Match]
Name=eth0

[Network]
DHCP=yes
IgnoreCarrierLoss=5 seconds

[DHCPv4]
RouteMetric=4096

[IPv6AcceptRA]
RouteMetric=4096
# stable IPv6 interface identifier across images
Token=prefixstable,8b4d3eba-43f1-461e-b582-7a782646afd5
```

On the test network that gives:

```
eth0   UP   192.168.1.20/24
           2a02:586:.../64  fe80::ad:32ff:fe01:ab6b/64
gateway    192.168.1.1
speed      100 Mbps, full duplex
```

The Ethernet controller is the internal RMII PHY (`meson8b-dwmac`), not a
USB adapter.

### DNS

`/etc/resolv.conf` is a **static file** with `8.8.8.8` and `1.1.1.1`.
`systemd-resolved` reports `resolv.conf mode: foreign`, which means it is
not managing that file. The DHCP-provided DNS (`192.168.1.1`) is known to
resolved for the link, but the static file wins for ordinary lookups.

This is deliberate: the vendor's DHCP client did not write `resolv.conf`,
so `scripts/prepare-stick.sh` wrote a working one. If you would rather have
resolved manage it and use the DHCP server's DNS:

```sh
rm -f /etc/resolv.conf
ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
systemctl restart systemd-resolved
```

That is a recipe, not the default.

### Hostname

The hostname is `sc2` (`/etc/hostname`). The build now adds

```
127.0.1.1   sc2
```

to `/etc/hosts`, so `hostname -f` and `getent hosts sc2` work locally.
The vendor's `127.0.0.1 tvbox` line is left alone; it is harmless.

Without that line, `hostname -f` fails with `Name or service not known`,
and anything that wants the local FQDN breaks. It was missing in the
published v0.7-tty image; the next build carries the fix, and the smoke
test now checks it.

## A static address

Edit `/etc/systemd/network/10-eth0.network` and replace the `[Network]`
section:

```
[Match]
Name=eth0

[Network]
Address=192.168.1.50/24
Gateway=192.168.1.1
DNS=192.168.1.1
```

Then:

```sh
networkctl reload
networkctl reconfigure eth0
ip -br addr show eth0
```

**What was tested:** the mechanism, on `dummy0`, so the SSH session could
not drop. A `30-dummy0.network` with `Address=10.99.0.1/24` was applied
with `networkctl reload && networkctl reconfigure dummy0`; the address
appeared and `ping 10.99.0.1` worked; eth0 kept its DHCP address. The
config was then removed.

**What was not tested:** applying the exact `10-eth0.network` change live.
That drops the DHCP address, and if the static address is wrong the box
becomes unreachable. Do it with the serial console or the stick at hand.

## mDNS

mDNS is **off**: `resolvectl` shows `-mDNS` globally and on every link.
LLMNR is on.

To turn mDNS on:

```sh
mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nMulticastDNS=yes\n' > /etc/systemd/resolved.conf.d/10-mdns.conf
systemctl restart systemd-resolved
resolvectl mdns                 # Global: yes
resolvectl mdns eth0 yes        # the link can still report "no"
resolvectl mdns eth0            # Link 3 (eth0): yes
```

The per-link step is not cosmetic: after the global change, `eth0` still
reported `no` until it was set on the link. To make that survive a
restart, add the same setting to the network file:

```ini
# /etc/systemd/network/10-eth0.network
[Link]
MulticastDNS=yes
```

Avahi is **not installed**; this is resolved's built-in mDNS, enough for
`hostname.local` lookups between machines that implement it.

**Tested:** the global and per-link settings changed as above, and
`resolvectl status eth0` showed `+mDNS` while they were on. **Not
tested:** an actual name resolution against a second mDNS peer — there is
none on the test network.

## WiFi

**v0.7.2 does not bring up WiFi. v0.7.3-wifi does**, as six loadable
modules. This section records what was wrong, what the working stack is and
how it is shipped.

### The chip is a Realtek RTL8822CS

The module is an **Earda** part (MAC `ac:f4:2c:a9:d8:47`) carrying a
**Realtek RTL8822CS**. The SDIO client identifies itself as
`024c:c822` (`sdio:c07v024CdC822`) at
`/sys/bus/sdio/devices/mmc1:0001:1`.

Earlier documentation called this a Broadcom BCMDHD part. That was wrong.
The DTB's `wifi@1` node says `compatible = "brcm,bcm4329-fmac"`, and the
vendor kernel logs `bcmdhd_init_wlan_mem` / carries `dhd_static_buf`, but
those are Amlogic's generic SDIO-WiFi glue used across modules; neither
identifies the chip. The vendor's own 6.18 image ships
`drivers/xtra/rtl8822cs/8822cs.ko` (Realtek driver v5.12.1.1) and the
`rtw88` 8822c firmware, which agrees with the SDIO ID.

### Two bugs on the 5.4 vendor kernel

1. **The initial SDIO scan was suppressed.** `meson-gx-mmc` set
   `mmc->rescan_entered = 1` for SDIO hosts, so the host probed and then
   never ran `mmc_rescan` after `aml_wifi` powered the chip. Rebinding the
   host or pulsing the power line therefore found nothing. Setting
   `rescan_entered = 0` lets the initial scan run; the card then
   enumerates as `mmc1: new UHS-I speed SDR104 SDIO card at address 0001`.
2. **There was no SDIO client driver.** The vendor 5.4 tree only has
   PCIe `rtw88` (8822BE/CE). The lwfinger `rtw88` backport supports the
   8822CS over SDIO. Built against a config matching the running kernel
   (plus `CONFIG_MAC80211=m`), its five modules probe the card, download
   firmware, and create `wlan0`.

The working stack autoloads at boot and `iw dev wlan0 scan` returns BSS
entries without wedging the kernel. A WPA2 association was tested and
works; a sixth module, `ccm.ko`, is required (the vendor config omits
`CONFIG_CRYPTO_CCM`, so the first handshake failed at key installation).
Once CCM is loaded the box associates, gets DHCP, and passes traffic. A
cold boot reconnects automatically.

The userspace setup is:

- `wpa_supplicant@wlan0.service` enabled and the generic
  `wpa_supplicant.service` disabled, so two supplicants do not contend
  for `wlan0`;
- `/etc/wpa_supplicant/wpa_supplicant-wlan0.conf` (the vendor image makes
  this a symlink to `/root/wpa_supplicant.conf`) containing the network
  block. Keep it mode 0600;
- `/etc/modules-load.d/rtw88.conf` listing `ccm` and `rtw_8822cs`;
- `systemd-networkd` then applies `/etc/systemd/network/20-wlan0.network`
  and does DHCP. Ethernet remains the preferred default route.

### Status in the current tree

The current tree carries both fixes:

- `patches/0011-sdio-run-initial-rescan.patch` lets the 5.4 host run its
  initial scan after `aml_wifi` powers the chip;
- the image ships the in-tree `mac80211.ko` and `ccm.ko`, plus the four
  lwfinger rtw88 modules, from `kernel/modules/`.

The board DTB enables `sdio@fe088000` (`status = "okay"`, 4-bit,
`non-removable`, SDR104) with a `wifi@1` child; `aml_wifi` drives
`GPIOX_6` high and the SDIO pins are muxed to the host. On the development
box the stack cold-boots, enumerates as
`mmc1: new UHS-I speed SDR104 SDIO card`, creates `wlan0`, associates to
`TP-Link_8860` and passes traffic. The same cold boot was repeated with the
stripped module set that ships in the image.

`/etc/systemd/network/20-wlan0.network` does DHCP with
`RouteMetric=4096`, so Ethernet wins whenever both are up.

**v0.7.3-wifi ships this stack.** The image was written to a USB stick and
verified byte-for-byte, booted from the stick (24/24 smoke checks),
installed to the eMMC, and cold-booted from the eMMC with the stick removed
(24/24 smoke checks again). The `iw scan` warning is lifted for this stack:
it returns BSS entries and does not wedge the kernel.

## Bluetooth

The kernel has `bt_init`, `hci_uart` and the H4 protocol registered. There
is no `/sys/class/bluetooth` device and no Bluetooth tooling installed.
**Unverified.** Treat it as absent until someone tests a module and an
adapter.

## Security

The published password is `root` / `tvbox`, and root login over SSH is
enabled. Change it with `passwd` before the box is on a network you do not
trust. See the README's security note.

---

## Last verified

**2026-10-23**, Ethernet and the base network on the eMMC system
(`/dev/mmcblk0p2`):

- eth0 DHCP via systemd-networkd: `192.168.1.20/24`, gateway
  `192.168.1.1`, 100 Mbps full duplex, IPv4 and IPv6;
- `resolv.conf` static (`8.8.8.8`, `1.1.1.1`), mode `foreign`;
- `hostname -f` → `sc2` after the `/etc/hosts` fix; `getent hosts
  sc2` → `127.0.1.1`;
- static-address mechanism tested on `dummy0` (address applied, ping
  worked, eth0 unaffected); the exact eth0 change not applied live;
- mDNS off; the global + per-link recipe was applied and reverted, and
  `eth0` showed `+mDNS` while it was on; no second mDNS peer for an
  end-to-end lookup.

**2026-10-29**, WiFi on the development box after a cold boot of the new
kernel and the stripped module set:

- the SDIO card enumerates as `mmc1: new ultra high speed SDR104 SDIO
  card at address 0001`;
- `rtw_8822cs` loads firmware 9.9.6 (WOW 9.9.4) and creates `wlan0`
  with MAC `ac:f4:2c:a9:d8:47`;
- `wpa_supplicant@wlan0.service` is active and associates to
  `TP-Link_8860` with `PTK=CCMP GTK=TKIP`;
- `iw dev wlan0 link` shows the association and `ping 1.1.1.1` passes;
- `iw dev wlan0 scan` returns BSS entries without wedging the kernel.

Bluetooth is still unverified: the kernel has `bt_init`, `hci_uart` and
H4 registered, but there is no `/sys/class/bluetooth` device and no
Bluetooth tooling installed.
