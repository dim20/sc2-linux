# `dist/` — local build artefacts

`dist/` holds build outputs, not source. Everything here is gitignored
except `SHA256SUMS` and this file.

The names are **not** all release names. Read this table before using a
file:

| File | What it actually is |
|---|---|
| `sc2-linux-stick.img.gz` | the **v0.8.0-docker** image. This is the "current" asset name the README tells a user to download; the file it names changes from release to release. It is the same bytes as `v0.8.0-docker-img.gz`. |
| `v0.8.0-docker-img.gz` | the same bytes as `sc2-linux-stick.img.gz`, kept under a release-named copy so it cannot be mistaken for another release. |
| `linux-image-sc2_5.4.210-sc2-docker1_arm64.deb` | the v0.8.0-docker kernel as an installable package (`Package: sc2-kernel`, version `5.4.210-sc2-docker1`), with rollback. |
| `v0.7.3-wifi-img.gz` | the **v0.7.3-wifi** image, kept as the previous release. |
| `linux-image-sc2_5.4.210-sc2-wifi2_arm64.deb` | the v0.7.3-wifi kernel package (`Package: sc2-kernel`, version `5.4.210-sc2-wifi2`). |
| `v0.7.2-img.gz` | the **v0.7.2-sc2** image, kept as the release before that. |
| `linux-image-sc2_5.4.210-sc2_arm64.deb` | the v0.7.2-sc2 kernel package (`Package: sc2-kernel`, version `5.4.210-sc2`). |
| `eon-s905x4-linux-stick.img.gz` | the **v0.7.1-tty** image under its old "current" name. |
| `v0.7.1-tty-img.gz` | the same bytes as `eon-s905x4-linux-stick.img.gz`, under its release name. |
| `v0.7.1-tty-kernel.deb` | the v0.7.1-tty kernel package, under the old `eon-kernel` name. Byte-identical to the v0.7-tty package. |
| `v0.7-tty-img.gz` | the **v0.7-tty** image (`c5c148cf…`). |
| `v0.7-tty-kernel.deb` | the v0.7-tty kernel package; the same bytes as `v0.7.1-tty-kernel.deb`. |
| `Image` | the kernel (`ee6433ad…`), the one v0.8.0-docker ships. |
| `sc2_s905x4_ah212_linux.dtb` | the device tree (`d1e2267f…`), unchanged since v0.7. |
| `v0.6-desktop-img.gz` | the **v0.6** image (`934d3969…`). Despite the name it contains no desktop — only `tasksel`. See [`../docs/releases/errata-v0.6.md`](../docs/releases/errata-v0.6.md) Correction 4. |
| `v0.5-manifest-img.gz` | the **v0.5** image (`7c6147cb…`). It is not v0.5.1. |
| `inventory.txt`, `TEST-CHECKLIST.md` | produced by the v0.8.0-docker build. |
| `SHA256SUMS` | covers the v0.8.0-docker assets above. Verify with `sha256sum -c SHA256SUMS` from this directory. |

## What is missing, and why

- **The v0.5.1 image is not here.** Its published SHA256 is
  `998899…` (see `docs/releases/v0.5.1-units.md`); it was built elsewhere
  and is not stored in this working tree.
- **No checksum for `v0.5-manifest-img.gz`** in `SHA256SUMS`: that file
  predates the current sums file. Its hash is
  `7c6147cbb469cf269d5ae4cc2cfc3c383f4fee7f90111442470099369a8f9472`.

## Which one should I use?

For containers, **v0.8.0-docker** is the current release. Its kernel
package was installed on the development box and cold-booted, and Docker
26.1.5 ran `hello-world` and fetched over the bridge network. The stick
image is built and manifest-checked; writing it to a stick and booting it
is the remaining hardware gate, listed in `TEST-CHECKLIST.md`.

The kernel package carries no modules. The WiFi stack and the optional
filesystems (`btrfs`, `xfs`, `cifs`, `nfs`, `dm-thin`, `nbd`) ship in the
OS image under `/lib/modules/<uname -r>/extra/`. A kernel-package update
must be paired with the matching module set from the same release.

The image contains a credential-free `/root/wpa_supplicant.conf` template,
so add your own network before expecting a WiFi association.

Debian 13's `iptables-nft` cannot append rules on this 5.4 kernel. Before
starting Docker, point `iptables` and `ip6tables` at the legacy
alternatives; the release note has the exact commands.
