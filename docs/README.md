# Documentation

Start here. This page is the map, and it says which document is the
authority when two of them disagree.

The product is **SC2 Linux**: Debian on the EON SDSTB02 (Amlogic
S905X4 / SC2) set-top box, booting unattended from eMMC with a text
console on HDMI and SSH over the network. It is meant to be a small
general-purpose Linux computer. It is not a media centre.

---

## Where to start

| You want to… | Read |
|---|---|
| use the box | [`../README.md`](../README.md) — status, quick start, safety |
| understand what is true right now | this page, then [`../README.md`](../README.md) |
| know what is planned | [`development.md`](development.md#roadmap) |
| see what works and what does not | [`limitations.md`](limitations.md) |
| understand the board and its boot chain | [`hardware.md`](hardware.md) |
| build, verify or release an image | [`development.md`](development.md) |
| fix a box that will not boot | [`recovery.md`](recovery.md) |
| start from an untouched or OTA-upgraded box | [`factory-box-install.md`](factory-box-install.md) — serial console, manual USB boot, install to eMMC |
| rebuild the kernel or the image | [`build-the-kernel.md`](build-the-kernel.md), [`kernel-package.md`](kernel-package.md) |
| understand the display path | [`framebuffer.md`](framebuffer.md) |
| understand storage, TRIM and eMMC wear | [`storage.md`](storage.md) |
| understand timekeeping and the clock fallback | [`time.md`](time.md) |
| understand power, reboots and HDMI hotplug | [`power.md`](power.md) |
| understand networking, static IP, mDNS and WiFi | [`network.md`](network.md) |
| understand thermals and sustained load | [`thermal.md`](thermal.md) |
| understand audio and the HDMI mixer trap | [`audio.md`](audio.md) |
| avoid a trap we already hit | [`troubleshooting.md`](troubleshooting.md), [`lessons.md`](lessons.md) |
| know the licensing | [`licensing.md`](licensing.md) |

## The hierarchy of truth

When two sources disagree, the higher one wins.

| Rank | Source | What it is |
|---|---|---|
| 1 | **The shipped artefact** (`dist/`, the release assets) | What the image, kernel and DTB actually are. Checked with `sha256sum -c SHA256SUMS`. |
| 2 | **`manifest/`** | Machine-checked claims about the image, enforced by `scripts/check-image.sh`. Anything that is not enforced belongs in `vendor-leftovers.txt` as a record. |
| 3 | **`docs/releases/`** | The release notes. Immutable once published; corrections are added as **errata**, not by rewriting them. |
| 4 | **`README.md`** | The current product and status. Always describes the latest release. |
| 5 | **`docs/`** (this directory) | How-to and reference. Behaviour claims carry a `Last verified` date where they have been checked. |
| 6 | **The development repository's `plan/PLAN-OF-RECORD.md`** | Decisions, priorities and the to-do list. The only current plan; everything else in `plan/` is archived. |
| 7 | **`notes/`** | A chronological lab notebook. Allowed to be wrong; never a source of truth without a matching document. |

## Three rules

1. **A claim about behaviour needs a date and a witness** — a log line, a
   command's output, or "observed on the display". "It works" is not
   evidence.
2. **Supersede, don't delete.** A finding that is no longer true is marked
   `SUPERSEDED <date> — <what is true instead>`. Deleting it loses the
   reason the current design exists.
3. **One question, one answer.** If two documents answer the same
   question, one of them links to the other.

## What is true now (2026-10-29)

- The **latest release is v0.7.3-wifi**: the RTL8822CS WiFi stack as six
  loadable modules. It passed the stick boot, eMMC install and standalone
  eMMC boot gates. See
  [`releases/v0.7.3-wifi.md`](releases/v0.7.3-wifi.md).
- The **text console on HDMI works** and the USB stick is both installer
  and recovery key.
- **WiFi works** once credentials are added; the image ships a
  credential-free template. See [`network.md`](network.md).
- The **desktop is frozen** and returns, if at all, as an optional package
  set. There is no GPU-accelerated 3D.
- The kernel is a **vendor 5.4.210 tree**; it does not track upstream
  security updates.
- An **untouched factory box** may need `run recovery_from_udisk` once on
  the serial console. See [`factory-box-install.md`](factory-box-install.md).
- The full caveat list is in [`limitations.md`](limitations.md).

## How to verify a claim

```sh
# the image and the kernel are what the release says
sha256sum -c SHA256SUMS

# the image satisfies the manifest
scripts/check-image.sh --rootfs /path/to/tree --p1 /path/to/p1.img

# the published patches reproduce the tree the kernel is built from
# (needs a clean CoreELEC amlogic-5.4.210 checkout; see build-the-kernel.md)
scripts/check-patches.sh patches /path/to/linux-amlogic

# the documentation does not contradict itself
scripts/check-docs.sh
```

On the box, the evidence for the console is:

```sh
dmesg | grep -E 'Taking over console|Console: switching'
cat /proc/consoles        # lists the serial port; a VT appears only with console=tty0
```
