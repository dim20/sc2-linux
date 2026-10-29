# Development and repository layout

This page is for people building, modifying or releasing the image. End
users do not need it.

## Verifying the box after a change

`scripts/smoke-test.sh` is installed into `/root` by the build. After a
kernel or image change, run it on the box:

```
/root/smoke-test.sh
```

It checks the branding, the console arrangement, failed units, the absence
of a desktop, `dpkg --audit`, `apt-get check`, the kernel and DTB on the
boot partition against their expected hashes, and the running system. It
is read-only. Exit status 0 means everything passed; `--quiet` prints only
the failures.

**The published v0.7-tty image predates this script; v0.7.1-tty and
later carry it**, so on the older download the file is not there. Copy
`scripts/smoke-test.sh` to `/root` on the box if you want it; every image
built from this tree afterwards carries it.

## Repository layout

```
.github/workflows/checks.yml     CI: check-docs.sh + script syntax on every push
dist/                           build artefacts; README.md says which file is which release
docs/
  README.md                       the map of the documentation, and the rules
  releases/                       the release notes, immutable, plus errata
  limitations.md                  the full list of known gaps and caveats
  hardware.md                     the board, other S905X4 boxes, and the boot chain
  development.md                  this page: layout, manifest, roadmap
  install-image-requirements.md   what the image must get right, and why
  build-the-kernel.md             how the kernel is built; required for GPL compliance
  recovery.md                     unlock / ADNL / fastboot, and how to get back in
  kernel-package.md               the kernel as an installable package, and rollback
  framebuffer.md                  how the display really works, and the tools that show it
  storage.md                      partitions, TRIM, eMMC wear and free space
  time.md                         the clock fallback, chrony and what to expect
  power.md                        reboot, poweroff and HDMI hotplug, with evidence
  network.md                      DHCP, static IP, DNS, mDNS, WiFi and Bluetooth
  thermal.md                      a sustained-load run, temperatures and throttling
  audio.md                        HDMI audio, the mixer trap, and how to test it
  troubleshooting.md              the traps, written down so you can avoid them
  lessons.md                      engineering lessons from this project
  licensing.md                    licences, credits, provenance
manifest/
  required-packages.txt           packages the image must contain
  forbidden-packages.txt          packages it must not (no desktop, no display manager)
  required-paths.txt              files it must contain, with their modes
  forbidden-paths.txt             files it must not contain
  forbidden-strings.txt           text that must not appear in the login files
  required-units.txt              systemd units it must have enabled
  masked-units.txt                units that must be masked
  global.txt                      rules that apply to the whole rootfs
  boot-artifacts.txt              what must be at the root of the boot partition
  kernel-package.txt              what the kernel package must contain
  inventory.txt                   every file in the image, with owner, mode, size, checksum
  provenance.txt                  where the rootfs came from, and what we changed
  vendor-leftovers.txt            what is still the vendor's, and what it costs
scripts/
  install-to-emmc.sh              provision eMMC from a running box
  rescue-init.sh                  minimal init — the fallback if systemd fails
  find-env.sh                     locate the U-Boot environment (read-only)
  prepare-stick.sh                make a stick a self-contained live system
  grow-stick.sh                   extend a stick's rootfs into leftover space
  smoke-test.sh                   one command after every kernel or image change
  build-stick-image.sh            build the published .img.gz from a rootfs tree
  build-kernel-package.sh         build the kernel as an installable .deb
  check-image.sh                  check an image (and a kernel package) against the manifest
  check-patches.sh                prove the published patches build the tree
  check-docs.sh                   prove the documentation does not contradict itself
  inventory.sh                    write down what is in an image, and diff it
  release.sh                      build a release, check it, print the test checklist
  write-stick-image.sh            write it to a stick and verify it byte-for-byte
  deploy-kernel.sh                send a kernel to a box and prove it arrived
  overrides/                      files the image build substitutes for the vendor's
  fbtools/                        small static tools that measure the framebuffer
  stick/                          the aml_autoscript uImage and its source
  uart/                           serial console tooling
kernel/
  sc2_s905x4_ah212_linux.dtb
  SHA256SUMS
  VERSION                         the project's release label for the kernel package
patches/                          local patches applied to the vendor tree
notes/journal.md                  chronological project log
```

The kernel `Image` is a **release asset**, not in the repo — it is 34 MB.
Download it from the Releases page and check it against
`kernel/SHA256SUMS`.

## The manifest

`manifest/` is the answer to "what is in this image, and what must be in
it". It is **data**, not code: plain text files listing what the image must
contain, what it must not, and which of the vendor's leftovers are still
there.

The build checks the image it has just produced against it and **refuses
to produce an image that fails**. `scripts/check-image.sh` can also be run
by hand against any rootfs tree:

```sh
scripts/check-image.sh --rootfs /path/to/tree --p1 /path/to/p1.img
```

Two things follow from this that are worth knowing:

* **The build does not modify its input.** It copies the rootfs tree to a
  scratch directory first and works on the copy, so the tree you give it
  is exactly the tree you still have afterwards — and building twice gives
  the same result as building once.
* **The build is reproducible.** The filesystem's timestamps and UUIDs and
  the gzip header are all pinned, so the same tree gives the same image,
  byte for byte.

`manifest/inventory.txt` lists every file in the image with its owner,
mode, size and checksum. Every build prints the difference against it, so
"what changed in this release" is a diff rather than a surprise.

**The manifest states what is true.** A rule that is not enforced does not
belong there; it belongs in `vendor-leftovers.txt` as a record. The vendor
kernel modules are recorded there, with their cost, because they are still
in the image — and the same is true of the vendor's branding packages.

## Roadmap

The phases are the plan of record's, not a new numbering:

| | Phase | State |
|---|---|---|
| H | documentation and housekeeping | **done** (2026-10-23), including the fresh-eyes review |
| P0 | **v0.7-tty**: finish the base and release it | **done** |
| P1 | a solid base: audio, power, thermal, storage, smoke test | **next** |
| P2 | desktop, splash, and the boot experience | **frozen** |
| P3 | hardware: WiFi, a second box, other devices | WiFi **done** in v0.7.3; other devices planned |

The detailed plan, including what "done" means for each item, is
`plan/PLAN-OF-RECORD.md` in the development repository (it is not part
of this repository). The vblank-event work, an upstream `meson` port,
Wayland and FbTerm are parked, each with its reason written down there.
