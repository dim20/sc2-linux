# Errata for v0.7.2-sc2

## Correction 1: the eMMC install path was verified

The v0.7.2-sc2 release note said the eMMC install path was not re-run for
this build. It was in fact installed to the eMMC during the release work,
and the box cold-booted without the stick and worked correctly. This
corrects the "not re-run" statements in the note's "What is still broken"
and "Last verified" sections.

The 24/24 smoke-test result recorded in the note was observed from the
stick boot. If an eMMC smoke-test result is wanted as well, run
`/root/smoke-test.sh` after the cold boot and add it here.

## Correction 2: the WiFi chip is Broadcom BCMDHD, not Realtek

The release note (and several older documents) called the WiFi chip an
"RTL8812CS" with a `rtl88x2cs` driver. That attribution is **wrong and
was never supported by a saved log.** The vendor BSP is **Broadcom
BCMDHD** throughout:

- the board DTB's `wifi@1` child is `compatible = "brcm,bcm4329-fmac"`;
- the `aml_wifi` node carries `dhd_static_buf`;
- `drivers/amlogic/wifi/dhd_static_buf.c` in the 5.4 tree defines the
  Broadcom dhd release string `101.10.361.10 (wlan=r892223-20210623-1)`;
- every 5.4 and 6.18 boot logs `bcmdhd_init_wlan_mem`;
- the running kernel's own strings include `bcmdhd_init_wlan_mem`,
  `bcmdhd_mem_prealloc` and `dhd_static_buf`.

The module's MAC `ac:f4:2c:xx:xx:xx` resolves to **Earda**, which is the
module maker, not the chip vendor. The "Realtek" idea most likely came
from the vendor's `drivers/xtra` leftovers (MT7668, RTL8189/8821/8822),
which are for *other* board variants and are not this board's stack.

The practical status is unchanged: the shipped 5.4 kernel's SDIO host
probes but does not enumerate a card, and no SDIO client driver is
registered. The card **does** enumerate on the devmfc 6.18 kernel
(`mmc2: new UHS-I SDR104 SDIO card`, `serial/A2.log:319`), so the work
is a 5.4 BSP bring-up gap — port the vendor's `bcmdhd` SDIO glue — not a
search for a Realtek driver. See `docs/network.md` for the evidence.
