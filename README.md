# sc2-linux — SC2 Linux

Debian on the **EON SDSTB02** — an Amlogic **S905X4 / SC2** set-top box
with 2 GB LPDDR4 and 14.6 GB eMMC — turned into a small, silent,
general-purpose Linux computer.

The box ships with a locked Android bootloader and no way to boot anything
else. This project gets it to boot Debian from eMMC, unattended, with
nothing plugged in. The USB stick is the installer and the recovery key.

**Not affiliated with Amlogic, SDMC, or EON/NOVA.** "EON SDSTB02" and
"Amlogic S905X4" are used descriptively to identify the hardware.

---

## Current release

**v0.8.0-docker** (2026-11-09) — Docker/containerd/runc support. Kernel
package label `5.4.210-sc2-docker1`.

The kernel now creates container namespaces and cgroups, provides the
bridge/nftables plumbing and overlay2 storage, and fits the 48 MiB eMMC
BOOT partition by shipping the optional filesystems as loadable modules.
The kernel package was installed on the development box, rebooted, and
`docker run hello-world` plus a bridge-network fetch succeeded. The stick
image is built and manifest-checked; the stick boot itself is the remaining
hardware gate.

See [`docs/releases/v0.8.0-docker.md`](docs/releases/v0.8.0-docker.md) for
the full note, assets and hashes.

## Quick start

Download the current image from the
[Releases page](https://github.com/dim20/sc2-linux/releases/latest). The
file is `v0.8.0-docker-img.gz`; the same bytes are published as
`sc2-linux-stick.img.gz`.

Write it to a USB stick of 1 GB or more:

```sh
gunzip -c sc2-linux-stick.img.gz | sudo dd of=/dev/sdX bs=4M status=progress
sync
```

Plug the stick into the box and power-cycle it.

- **If the box already prefers USB**, it boots the stick automatically.
- **If the box is untouched or OTA-upgraded**, its U-Boot boots the eMMC
  first. You need the serial console once and the command
  `run recovery_from_udisk`. See
  [`docs/factory-box-install.md`](docs/factory-box-install.md).

Once booted, log in on HDMI or over SSH:

```sh
ssh root@<the address the box prints on the console>
```

The default login is `root` / `tvbox`. Change it with `passwd` before the
box is on a network you do not trust.

Install to the eMMC:

```sh
/root/install-to-emmc.sh
```

The installer verifies every artefact before and after writing, grows the
filesystem to fill the eMMC, and sets the U-Boot boot command. Remove the
stick and power-cycle to confirm standalone boot.

After a kernel or image change, run `/root/smoke-test.sh` on the box. Exit
status 0 means all checks passed.

## What works

| | |
|---|---|
| Boot | ✅ unattended from eMMC and from the USB stick |
| Console | ✅ text console on HDMI and serial; USB keyboard works |
| Network | ✅ Ethernet, DHCP, DNS, SSH |
| Audio | ✅ HDMI audio (the image selects `Tdm_C` at boot) |
| eMMC install | ✅ automated, verifies every write, sets the boot command |
| Kernel rollback | ✅ via the separate `sc2-kernel` package (`sc2-kernel-list` / `sc2-kernel-select`) |
| Clock | ✅ chrony with a persisted last-known-good fallback |
| WiFi | ✅ RTL8822CS driver, firmware and `wlan0`; add your own credentials (association tested on the development box) |
| Containers | ✅ Docker/containerd/runc: namespaces, cgroups, overlay2 and bridge networking (install `docker.io docker-cli`, then see the legacy-iptables note below) |
| Verification | ✅ 24-check smoke test shipped since v0.7.1-tty |

## What does not work, or needs care

| | |
|---|---|
| Desktop / 3D | ❌ no desktop and no GPU-accelerated 3D |
| WiFi credentials | ⚠️ not shipped; add a network block to `/root/wpa_supplicant.conf` |
| Bluetooth | ❌ unverified; no `/sys/class/bluetooth` device |
| Hardware coverage | ⚠️ tested on one board model only |
| Kernel updates | ⚠️ vendor 5.4.210 tree; no upstream security updates |
| Docker networking | ⚠️ Debian 13's `iptables-nft` cannot append rules on this 5.4 kernel; point `iptables`/`ip6tables` at the legacy alternatives before starting Docker (see the release note) |
| Default password | ⚠️ `root` / `tvbox` is published; change it |
| Old images | ⚠️ images older than v0.7 panic if `/dev/fb0` is opened |
| Factory boxes | ⚠️ some need the serial console once to boot the stick |

The detailed list is in [`docs/limitations.md`](docs/limitations.md).

## If it will not boot

1. **Plug the stick in.** It boots a complete Linux system with SSH even if
   the eMMC is broken.
2. **Use the U-Boot prompt.** With no stick attached, `bootdelay=3` gives a
   three-second window to interrupt and boot a different `init=`.
3. **Use DNL / fastboot** as the last resort.

The full ladder, including the UART wiring and recovery commands, is in
[`docs/recovery.md`](docs/recovery.md).

## Documentation

| You want to… | Read |
|---|---|
| use the box | this page |
| install from an untouched or OTA-upgraded box | [`docs/factory-box-install.md`](docs/factory-box-install.md) |
| fix a box that will not boot | [`docs/recovery.md`](docs/recovery.md) |
| configure network, static IP, mDNS or WiFi | [`docs/network.md`](docs/network.md) |
| see the full limitations and traps | [`docs/limitations.md`](docs/limitations.md), [`docs/troubleshooting.md`](docs/troubleshooting.md) |
| understand the board and boot chain | [`docs/hardware.md`](docs/hardware.md) |
| know which version is installed | [`docs/kernel-package.md`](docs/kernel-package.md#which-version-is-installed) |
| build the kernel, image or release | [`docs/development.md`](docs/development.md), [`docs/build-the-kernel.md`](docs/build-the-kernel.md) |
| see what changed in each release | [`docs/releases/`](docs/releases/) |
| know the licences and provenance | [`docs/licensing.md`](docs/licensing.md) |
| browse the documentation map | [`docs/README.md`](docs/README.md) |

## Safety

- Read [`docs/recovery.md`](docs/recovery.md) before writing anything to a
  box.
- Provisioning overwrites the eMMC user area. Back up anything you care
  about.
- The default password is published. Change it before exposing the box.
- This project is not affiliated with Amlogic, SDMC or EON/NOVA.

## Licence

The scripts, documentation and notes are **MIT**. The kernel, device tree
and loaded modules are **GPL-2.0**; the Debian rootfs is GPL-2.0 and the
licences of its packages. Full detail and credits are in
[`docs/licensing.md`](docs/licensing.md).
