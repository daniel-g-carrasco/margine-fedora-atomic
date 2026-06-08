# ADR 0008 - Titanoboa ISO migration plan

**Status:** Proposed
**Date:** 2026-06-08
**Supersedes:** the earlier "keep BIB for now" ISO migration notes in
`margine-image`. This ADR is the planning source of truth for the next ISO
implementation.

## 1. Context - Why We Are Migrating

Margine's current install ISO path uses `osbuild/bootc-image-builder-action`
with an Anaconda ISO config. That path has been made to work, but it has also
produced the exact issues that make the ISO track hard to operate:

- The project lead reported that `margine-anaconda-iso-20260607` is about
  9 GB.
- The current BIB workflow is pinned both to a BIB action commit and a BIB
  container digest in `margine-image`:
  [workflow lines 51-54](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/.github/workflows/build-disk.yml#L51-L54).
- The workflow already works around GitHub runner disk pressure before BIB
  runs:
  [workflow lines 97-100](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/.github/workflows/build-disk.yml#L97-L100).
- The ISO Flatpak path had a prior install-time failure mode where the
  install completed but Flatpaks were absent; the current config documents the
  suspected `/tmp` tmpfs/OOM cause and the move to build-time baking:
  [iso-gnome.toml lines 236-240](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L236-L240).
- The current Anaconda config includes a custom partitioning contract,
  MOK import flow, Btrfs tuning, Flatpak rsync, and Anaconda module choices:
  [partitioning lines 34-70](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L34-L70),
  [bootc switch lines 72-78](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L72-L78),
  [MOK import lines 80-137](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L80-L137),
  [Btrfs tuning lines 139-218](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L139-L218),
  [Flatpak rsync lines 220-278](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L220-L278), and
  [Anaconda modules lines 291-303](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L291-L303).
- The project lead reported that on the first post-install reboot the MOK
  Manager prompt did not appear on Margine, while Bluefin did show the prompt
  on the same hardware. This is user-reported hardware behavior, not yet
  independently reproduced by this ADR.
- The project lead also named production pain around partitioning UX,
  Anaconda profile behavior, and installer ergonomics. Those are treated here
  as user-reported issues from the 2026-06-08 migration request; the exact UX
  failures still need to be reproduced during Phase 5 hardware testing.

The goal is not to "try Titanoboa" as a vague replacement. The goal is to
move the ISO workflow to the current Titanoboa contract while preserving
Margine's installer behavior, Secure Boot behavior, baked Flatpaks, and
bootc origin.

Titanoboa is the right candidate because Universal Blue's current ISO repos
have moved their live ISO planning around it. Bluefin's ISO repo explicitly
says it builds Bluefin ISOs using Anaconda and Titanoboa:
[README lines 1-12](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/README.md#L1-L12).
Bazzite's current ISO workflow also builds a live/payload image and invokes a
Titanoboa revamp branch:
[Bazzite workflow lines 123-143](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/.github/workflows/build_iso.yml#L123-L143).

## Decision

Migrate the Margine Anaconda ISO from BIB to Titanoboa by introducing a
dedicated Margine live-environment image. That image will embed Titanoboa's
current required files, the Anaconda live installer, Margine's generated
Kickstart/defaults, BAKE Flatpaks, the install payload image, and the
`/usr/lib/bootc-image-builder/iso.yaml` contract. The GitHub workflow will
build that live-environment image and call Titanoboa with only the current
supported action inputs: `image-ref` and `iso-dest`.

Keep Anaconda for the first Titanoboa migration. Titanoboa is an ISO builder,
not an installer selection mechanism. The current upstream examples and
active consumers that still use Titanoboa place the installer inside the live
image. Bazzite and Bluefin both use Anaconda-style live installer content in
their ISO images:
[Bazzite postrootfs lines 10-16](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_postrootfs.sh#L10-L16) and
[Bluefin configure lines 56-68](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/iso_files/configure_iso_anaconda.sh#L56-L68).

Do not copy Bluefin's current workflow literally. Bluefin's source workflow
still passes old Titanoboa inputs:
[Bluefin workflow lines 168-176](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/.github/workflows/reusable-build-iso-anaconda.yml#L168-L176).
Current Titanoboa action metadata only declares `image-ref` and `iso-dest`:
[Titanoboa action lines 5-16](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml#L5-L16).

Use Bazzite's current source and Titanoboa's own Bazzite example as the main
implementation references for the first prototype. Bazzite's current workflow
builds a payload container and calls a Titanoboa revamp branch with only
`image-ref` and `iso-dest`:
[Bazzite workflow lines 123-143](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/.github/workflows/build_iso.yml#L123-L143).

## 2. Titanoboa Today - Verified Upstream Behaviour

Research date: 2026-06-08. Upstream commit inspected:
[`ublue-os/titanoboa@5c457c3d0518bd17e754be0fd98a60d29d26abb4`](https://github.com/ublue-os/titanoboa/tree/5c457c3d0518bd17e754be0fd98a60d29d26abb4).

Titanoboa currently describes itself as a bootc installer that creates
bootable ISOs from bootc images:
[README lines 3-11](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/README.md#L3-L11).
Its current direction is to remove external installer dependency and move all
customization into the container:
[README line 15](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/README.md#L15) and
[README lines 35-42](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/README.md#L35-L42).

Current action API:

| Name | Kind | Required | Type | Default | Description | Source |
| --- | --- | --- | --- | --- | --- | --- |
| `image-ref` | input | yes | implicit string; no `type` key in `action.yml` | none | Container image reference used to build the ISO | [action lines 5-8](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml#L5-L8) |
| `iso-dest` | input | no | implicit string; no `type` key in `action.yml` | `${{ github.workspace }}/output.iso` | Output ISO path | [action lines 9-12](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml#L9-L12) |
| `iso-dest` | output | n/a | implicit string | n/a | Output ISO path | [action lines 13-16](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml#L13-L16) |

The composite action exports `DESTINATION_PATH` and `TITANOBOA_CTR_IMAGE`,
runs `main.sh`, then moves the produced ISO to `iso-dest`:
[action lines 21-42](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml#L21-L42).

`main.sh` can run inside Titanoboa's own container or outside it. Inside a
container, it requires mounted `/output`, `/usr/lib/containers/storage`, and
`/rootfs`; outside a container, it launches `quay.io/fedora/fedora:latest`
with the target image mounted at `/rootfs`:
[main.sh lines 30-76](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/main.sh#L30-L76).
The source tree inspected at the current commit has no `Justfile`; the top
level contains `Containerfile`, `README.md`, `action.yml`, `build_iso.sh`,
`examples/`, and `main.sh`. This is a verified repository-tree negative
result, so there is no line-numbered source file for the absent `Justfile`.

Current image contract:

- The live image must include `/usr/lib/bootc-image-builder/iso.yaml`.
  Titanoboa exits if it is missing:
  [build_iso.sh lines 18-22](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/build_iso.sh#L18-L22).
- `iso.yaml` supports a label and GRUB entries:
  [README lines 44-58](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/README.md#L44-L58).
- Titanoboa expects a kernel/initramfs under `/usr/lib/modules/*`, UEFI
  binaries under `/boot/efi/EFI/$VENDOR`, and GRUB modules under
  `/usr/lib/grub/i386-pc`:
  [README lines 75-83](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/README.md#L75-L83).
- `build_iso.sh` copies the live initramfs and kernel into
  `/images/pxeboot`, copies EFI assets, generates `grub.cfg`, creates a FAT
  EFI image, and uses `xorriso` to produce the ISO:
  [build_iso.sh lines 24-94](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/build_iso.sh#L24-L94).
- `build_iso.sh` reads the ISO label directly from `.label`; GRUB timeout
  defaults to 10 and default menu entry defaults to 0 if those fields are
  omitted:
  [build_iso.sh lines 24 and 40-43](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/build_iso.sh#L24-L43).
- The root filesystem becomes `/LiveOS/squashfs.img`, produced from the
  source container rootfs with `sysroot` and `ostree` excluded:
  [build_iso.sh line 16](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/build_iso.sh#L16).
- The containerized runner installs `squashfs-tools`, `xorriso`, `yq`,
  `mtools`, and `dosfstools`:
  [build_iso.sh lines 5-12](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/build_iso.sh#L5-L12) and
  [Containerfile lines 1-22](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/Containerfile#L1-L22).

The current Bazzite example in Titanoboa is useful because it shows the
container-internal customization model after the breaking change. It installs
Flatpaks, pulls the install payload, installs `dracut-live`, enables livesys
services, installs `grub2-efi-x64-cdboot`, copies EFI assets, and writes
`iso.yaml`:
[example build.sh lines 16-83](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/examples/bazzite/src/build.sh#L16-L83).
Its `iso.yaml` adds the live boot arguments:
[example iso.yaml lines 1-7](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/examples/bazzite/src/iso.yaml#L1-L7).
It also swaps the live ISO kernel to a vanilla Fedora kernel for Secure Boot:
[example preinitramfs lines 5-21](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/examples/bazzite/src/titanoboa_hook_preinitramfs.sh#L5-L21).

Live-environment package conclusion:

- Titanoboa itself does not check RPM package names. It checks for outcomes:
  `iso.yaml`, kernel, initramfs, EFI assets, GRUB modules, and the files used
  to build the SquashFS/ISO:
  [README lines 75-83](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/README.md#L75-L83) and
  [build_iso.sh lines 18-94](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/build_iso.sh#L18-L94).
- The practical packages to produce those outcomes are verified from current
  Bazzite/Titanoboa examples: `dracut-live`, `livesys-scripts`,
  `grub2-efi-x64-cdboot`, and Anaconda live packages:
  [Titanoboa example lines 51-83](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/examples/bazzite/src/build.sh#L51-L83),
  [Bazzite build lines 51-85](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/build.sh#L51-L85), and
  [Bazzite postrootfs lines 10-16](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_postrootfs.sh#L10-L16).
- Fedora package metadata matches those roles: `dracut-live` provides live
  image capabilities and includes `dmsquash-live` modules; `livesys-scripts`
  auto-configures live media during boot; `grub2-efi-x64-cdboot` provides
  removable-media EFI boot files; `anaconda-live` contains live-installation
  files and dependencies. Sources:
  <https://packages.fedoraproject.org/pkgs/dracut/> lines 14-21,
  <https://packages.fedoraproject.org/pkgs/dracut/dracut-live/fedora-44.html> lines 43-78,
  <https://packages.fedoraproject.org/pkgs/livesys-scripts/livesys-scripts/> lines 6-17,
  <https://packages.fedoraproject.org/pkgs/grub2/grub2-efi-x64-cdboot/> lines 5-15, and
  <https://packages.fedoraproject.org/pkgs/anaconda/anaconda-live/> lines 6-17.

### Upstream Breaking Changes

GitHub API metadata was used for pull request titles, merge times, and latest
run status. GitHub API responses are primary-source metadata, but they are not
line-numbered files. Source-file facts below use line-numbered GitHub URLs.

The important Titanoboa breaking change is
[#138, `feat!: Only use container images as the only source of truth`](https://github.com/ublue-os/titanoboa/pull/138),
merged 2026-05-19. Its merge commit is the currently inspected upstream head,
`5c457c3d0518bd17e754be0fd98a60d29d26abb4`. Before that change, the action
accepted many workflow-level customization inputs:
`image-ref`, `livesys`, `compression`, `hook-post-rootfs`,
`hook-pre-initramfs`, `iso-dest`, `flatpaks-list`, `container-image`,
`add-polkit`, `kargs`, and `builder-distro`:
[pre-#138 action lines 5-49](https://github.com/ublue-os/titanoboa/blob/3921b27f94b6f807cc0aa8768bacf2856245dc4b/action.yml#L5-L49).
After #138, the action declares only `image-ref`, `iso-dest`, and the
`iso-dest` output:
[post-#138 action lines 5-16](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml#L5-L16).

That is why Margine cannot port the current BIB config by adding more action
inputs. The live environment has to contain the ISO config and installer
customizations.

The earlier breaking change is
[#38, `feat!: pass hooks as script paths`](https://github.com/ublue-os/titanoboa/pull/38),
merged 2025-03-25. Before #38, the post-rootfs hook was described as hook
content and passed through a heredoc/file descriptor:
[pre-#38 lines 17-20](https://github.com/ublue-os/titanoboa/blob/0b8cdd5e7acbbb1db41700f53f82e281b9262abf/action.yml#L17-L20) and
[pre-#38 lines 60-65](https://github.com/ublue-os/titanoboa/blob/0b8cdd5e7acbbb1db41700f53f82e281b9262abf/action.yml#L60-L65).
After #38, the hook is a path:
[post-#38 lines 17-21](https://github.com/ublue-os/titanoboa/blob/e15da21447e277bba4166472ca28936e6a820f1f/action.yml#L17-L21) and
[post-#38 lines 62-65](https://github.com/ublue-os/titanoboa/blob/e15da21447e277bba4166472ca28936e6a820f1f/action.yml#L62-L65).
#138 then removed those hook inputs from the action API entirely.

Recent merged Titanoboa PRs inspected through the GitHub API:

| PR | Merged | Title | Migration relevance |
| --- | --- | --- | --- |
| [#138](https://github.com/ublue-os/titanoboa/pull/138) | 2026-05-19 | `feat!: Only use container images as the only source of truth` | Breaks old action-input customization model. |
| [#135](https://github.com/ublue-os/titanoboa/pull/135) | 2026-01-04 | `feat: Add nomodeset entry to grub` | Shows GRUB entry behavior is still evolving. |
| [#129](https://github.com/ublue-os/titanoboa/pull/129) | 2025-11-09 | `fix: Replace broken links in almalinux` | Documentation/links. |
| [#126](https://github.com/ublue-os/titanoboa/pull/126) | 2025-10-04 | `Write end user documentation to get started building with Titanoboa` | Current docs direction. |
| [#115](https://github.com/ublue-os/titanoboa/pull/115) | 2025-07-07 | `fix: Change default timeout to 10` | Default boot menu behavior changed. |
| [#123](https://github.com/ublue-os/titanoboa/pull/123) | 2025-09-01 | `chore: Rename bootup entry in grub` | GRUB naming churn. |
| [#122](https://github.com/ublue-os/titanoboa/pull/122) | 2025-08-31 | `chore: use 'titanoboa_boot' as partition name` | ISO partition naming. |
| [#121](https://github.com/ublue-os/titanoboa/pull/121) | 2025-08-24 | `fix: remove 'dracut' and only install dracut-live` | Live initramfs dependency behavior. |
| [#119](https://github.com/ublue-os/titanoboa/pull/119) | 2025-08-11 | `chore(deps): update actions/checkout action to v5` | CI dependency. |
| [#118](https://github.com/ublue-os/titanoboa/pull/118) | 2025-08-07 | `feat: CentOS builder image` | Builder distro work. |
| [#114](https://github.com/ublue-os/titanoboa/pull/114) | 2025-07-05 | `feat: Add pre_initramfs hook` | Historical hook mechanism, removed from current action API. |
| [#109](https://github.com/ublue-os/titanoboa/pull/109) | 2025-06-18 | `chore: pin dependency action to hash` | CI hardening. |
| [#107](https://github.com/ublue-os/titanoboa/pull/107) | 2025-05-24 | `fix: allow non-app flatpaks to be listed in flatpak list` | Old Flatpak input model, removed by #138. |
| [#104](https://github.com/ublue-os/titanoboa/pull/104) | 2025-05-16 | `feat: leverage just dependency resolution` | Old Justfile era, removed by #138. |
| [#95](https://github.com/ublue-os/titanoboa/pull/95) | 2025-04-30 | `fix: grub.cfg.tmpl` | GRUB template work before current rewrite. |
| [#105](https://github.com/ublue-os/titanoboa/pull/105) | 2025-05-14 | `fix: set /etc/localtime so KDE Plasma has a clock` | Live environment polish. |
| [#74](https://github.com/ublue-os/titanoboa/pull/74) | 2025-04-20 | `fix: make it so kargs dont get read if they are "NONE"` | Old kargs input model, removed by #138. |
| [#72](https://github.com/ublue-os/titanoboa/pull/72) | 2025-04-20 | `feat: add customizeable kernel arguments for nvidia` | Old kargs customization model. |
| [#58](https://github.com/ublue-os/titanoboa/pull/58) | 2025-03-29 | `feat: Add log grouping at CI` | CI logging. |
| [#63](https://github.com/ublue-os/titanoboa/pull/63) | 2025-04-03 | `fix: only pull non-localhost images` | Image pull behavior. |

No `BREAKING` label was found on those API records; the breaking signal came
from the `feat!:` titles and the #138/#38 API metadata.

### Consumers And Reference Implementations

The unauthenticated GitHub dependents page reported 44 repositories depending
on `ublue-os/titanoboa`:
[dependents lines 185-189](https://github.com/ublue-os/titanoboa/network/dependents).
The visible page listed these consumer repositories:
[dependents lines 197-272](https://github.com/ublue-os/titanoboa/network/dependents):

`screwys/nocblue`, `Doraemon-kun/fedora-bluefin`,
`joshyorko/omarchy-bootc`, `bodnar-dev/aurora`, `rougeos/rouge`,
`clyphy/bazzite`, `kleinbem/bluefin`, `TofuTofuMe/ublue-tofu-t480`,
`dperson/silver-ublue`, `goodgollyholly/bluefin`, `Debug-Doctor/bazzite`,
`jillmnolan/bazzite`, `sanctari/bazzite`, `zelikos/zeliblue`,
`rsturla/eternal-main`, `alexiri/blueshift_v3`, `winblues/bluexp`,
`koyuawsmbrtn/freios`, `winblues/blue9`, and `winblues/blue95`.

This is not an exhaustive list of all 44 dependents; it is the complete set
visible during unauthenticated research.

### Bluefin

`projectbluefin/iso` says it builds Bluefin ISOs with Anaconda and Titanoboa,
with system Flatpaks, branding, and Secure Boot enrollment:
[README lines 1-12](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/README.md#L1-L12).
It also documents that LTS ISO work is disabled/broken and that only stable
promotion is safe:
[README lines 16-30](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/README.md#L16-L30).

Current Bluefin still calls `ublue-os/titanoboa@main` while passing old inputs:
[workflow lines 168-176](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/.github/workflows/reusable-build-iso-anaconda.yml#L168-L176).
Those extra inputs are not declared by current Titanoboa:
[Titanoboa action lines 5-16](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml#L5-L16).

Bluefin is still valuable for Anaconda content:

- It installs Anaconda live packages:
  [configure lines 56-68](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/iso_files/configure_iso_anaconda.sh#L56-L68).
- It writes Anaconda profile/defaults:
  [configure lines 72-145](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/iso_files/configure_iso_anaconda.sh#L72-L145).
- It rsyncs baked Flatpaks into the target:
  [configure lines 154-162](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/iso_files/configure_iso_anaconda.sh#L154-L162).
- It imports a Secure Boot key in `%post --nochroot`:
  [configure lines 164-193](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/iso_files/configure_iso_anaconda.sh#L164-L193).

Latest public Bluefin Actions metadata inspected through the GitHub API showed
the most recent `Build LTS-HWE Testing ISOs` run on 2026-06-08 completed with
failure, and several 2026-06-01 ISO runs also failed. Job log download returned
403 for this research session. ASSUMPTION (unverified): the stale Titanoboa
inputs may be related to some current Bluefin failures, but this ADR does not
claim that as verified root cause.

Bluefin release metadata inspected through the GitHub API showed the latest
stable release as `26.05.7-stable`, published 2026-05-11, with torrent and
checksum assets rather than raw ISO assets. The latest testing/LTS metadata
inspected was `26.05.9`, published 2026-05-18. This is API metadata and has
no line-numbered source file.

Issue search metadata for `projectbluefin/iso` did not find an open
Titanoboa/build-specific issue during this investigation; the query returned
the dependency dashboard issue with label `kind/automation`. This is API
metadata and has no line-numbered source file.

`ublue-os/main` was also inspected because Bluefin could have referenced a
shared Universal Blue workflow. The current reusable workflow is an image
build workflow with `image_version`, `image_name`, and `image_variant` inputs:
[ublue-os/main reusable-build lines 1-18](https://github.com/ublue-os/main/blob/65eaeb96b0bd90c4fa5f0f07e10f086f42585ec5/.github/workflows/reusable-build.yml#L1-L18).
The current changelog contains only historical ISO-generator material, including
a removal of the vestigial ISO generator:
[ublue-os/main changelog line 101](https://github.com/ublue-os/main/blob/65eaeb96b0bd90c4fa5f0f07e10f086f42585ec5/CHANGELOG.md#L101).

### Bazzite

Bazzite is the best current production reference. Its current workflow builds
a payload image and then calls a Titanoboa revamp branch with only
`image-ref` and `iso-dest`:
[workflow lines 123-143](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/.github/workflows/build_iso.yml#L123-L143).

Bazzite's installer image performs the relevant work inside the image:

- Installs Flatpaks and loads/pulls the install payload:
  [build.sh lines 20-30](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/build.sh#L20-L30).
- Runs the pre-initramfs hook, installs `dracut-live`, and builds a live
  initramfs:
  [build.sh lines 51-59](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/build.sh#L51-L59).
- Installs and enables livesys services:
  [build.sh lines 61-68](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/build.sh#L61-L68).
- Installs GRUB EFI CD boot support, copies EFI assets, and writes `iso.yaml`:
  [build.sh lines 70-85](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/build.sh#L70-L85) and
  [iso.yaml lines 1-10](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/iso.yaml#L1-L10).
- Makes `/var/lib/flatpak` available read-only for the live environment:
  [build.sh lines 91-125](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/build.sh#L91-L125).
- Uses Anaconda/Kickstart post scripts for bootc switch, MOK import, and
  user-facing docs:
  [postrootfs lines 37-123](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_postrootfs.sh#L37-L123),
  [postrootfs lines 126-153](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_postrootfs.sh#L126-L153), and
  [postrootfs lines 155-163](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_postrootfs.sh#L155-L163).
- Swaps the live ISO kernel to a vanilla Fedora kernel for Secure Boot:
  [preinitramfs lines 5-21](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_preinitramfs.sh#L5-L21).

Latest public Bazzite `Build Live ISOs` Actions metadata inspected through the
GitHub API showed a successful workflow-dispatch run on 2026-05-15. No
post-2026-05-19 public run was found in the latest runs inspected, so this ADR
does not treat Bazzite as post-#138 runtime proof. It is still the closest
current source reference because its workflow already matches the current
input shape.

### Aurora, Cosmic, And Other Universal Blue Repos

`ublue-os/aurora` was inspected at
[`90cd49c575780fde28efb56acdc3940b2d34095f`](https://github.com/ublue-os/aurora/tree/90cd49c575780fde28efb56acdc3940b2d34095f).
No current `titanoboa` workflow or file reference was found by repository
search during this investigation. This is a verified search negative result,
so there is no line-numbered source file for an absent workflow.

`ublue-os/cosmic` was inspected at
[`c498ba6720065a4bf76e8c49b087557f58ea1cff`](https://github.com/ublue-os/cosmic/tree/c498ba6720065a4bf76e8c49b087557f58ea1cff).
Its ISO workflow uses `jasonn3/build-container-installer@v1.2.3`, not
Titanoboa:
[cosmic workflow lines 85-100](https://github.com/ublue-os/cosmic/blob/c498ba6720065a4bf76e8c49b087557f58ea1cff/.github/workflows/build_iso.yml#L85-L100).

### Installer Choice

Titanoboa does not currently impose Anaconda, Calamares, or Readymade. Its
current source creates a live ISO from a container rootfs, `iso.yaml`, kernel,
initramfs, EFI assets, GRUB config, and SquashFS:
[README lines 44-83](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/README.md#L44-L83) and
[build_iso.sh lines 16-94](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/build_iso.sh#L16-L94).

Anaconda remains the right phase-1 installer because Margine already has an
Anaconda/Kickstart install contract and because the current Bluefin/Bazzite
references use Anaconda. Kickstart still supports `%post`, multiple `%post`
sections, and `--nochroot`, which is the mechanism Bluefin/Bazzite/Margine use
for MOK work and target filesystem work:
[pykickstart `%post` lines 8138-8149](https://pykickstart.readthedocs.io/en/latest/kickstart-docs.html#L8138-L8149) and
[pykickstart `--nochroot` lines 8169-8172](https://pykickstart.readthedocs.io/en/latest/kickstart-docs.html#L8169-L8172).

Calamares is not a phase-1 target. The GitHub mirror states that the project
has moved away from GitHub and identifies Calamares as a distribution-
independent installer framework:
[Calamares lines 149-151](https://github.com/calamares/calamares#L149-L151) and
[Calamares lines 327-336](https://github.com/calamares/calamares#L327-L336).

Readymade is also not a phase-1 target. Its docs describe it as a Fyra Labs
installer used by Ultramarine and related projects:
[Readymade docs lines 20-31](https://developer.fyralabs.com/rdms#L20-L31).
Its repository describes it as a replacement for Anaconda for Ultramarine and
tauOS, and a work-in-progress frontend for `systemd-repart`:
[Readymade repo lines 287-294](https://github.com/FyraLabs/readymade#L287-L294).
No inspected Titanoboa source path references Readymade.

## 3. Rules And Constraints - Invariants The Migration Must Not Break

These rules apply to the migration unless explicitly changed by a later ADR:

- **Use current Titanoboa API only.**
  Rationale: current `action.yml` declares only `image-ref`, `iso-dest`, and
  the `iso-dest` output:
  [action lines 5-16](https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml#L5-L16).
- **Keep Anaconda for the first migration.**
  Rationale: Margine already has Anaconda/Kickstart behavior to preserve, and
  Bluefin/Bazzite both use Anaconda-style live content:
  [Bluefin configure lines 56-145](https://github.com/projectbluefin/iso/blob/a4e89e2a93009035621ccd2713c9ff9595393857/iso_files/configure_iso_anaconda.sh#L56-L145) and
  [Bazzite postrootfs lines 10-123](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_postrootfs.sh#L10-L123).
- **Keep the installed Margine system on the CachyOS kernel path.**
  Rationale: `margine-image` signs the CachyOS kernel and modules with the
  Margine MOK and regenerates the initramfs:
  [install.sh lines 98-164](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/custom-kernel/install.sh#L98-L164) and
  [install.sh lines 447-467](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/custom-kernel/install.sh#L447-L467).
- **The live ISO must boot on Secure Boot hardware before Margine MOK is
  enrolled.**
  Rationale: Secure Boot validates bootloader and kernel signatures before the
  OS is running:
  [Red Hat Secure Boot lines 4267-4271](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_monitoring_and_updating_the_kernel/index#L4267-L4271),
  and an unsigned or untrusted kernel is prevented from booting:
  [Red Hat Secure Boot line 4284](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_monitoring_and_updating_the_kernel/index#L4284).
  Bazzite handles this by swapping the live ISO kernel to a vanilla Fedora
  kernel:
  [Bazzite preinitramfs lines 5-21](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_preinitramfs.sh#L5-L21).
- **Stage Margine MOK enrollment before the first installed-system boot.**
  Rationale: `mokutil --import` stages a key for MokManager after reboot, and
  the enrolled public key is required for signed-kernel trust:
  [Red Hat MOK lines 4588-4618](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_monitoring_and_updating_the_kernel/index#L4588-L4618) and
  [Red Hat prerequisites lines 4623-4628](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_monitoring_and_updating_the_kernel/index#L4623-L4628).
- **Do not ship the private MOK key in the live image.**
  Rationale: `margine-image` currently consumes MOK secrets through BuildKit
  secret mounts:
  [Containerfile lines 31-46](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/Containerfile#L31-L46).
  The live image needs the public DER and enrollment password path, not the
  private signing material.
- **Keep the bootc install origin stable.**
  Rationale: the current install flow switches the target to
  `ghcr.io/daniel-g-carrasco/margine:stable`:
  [iso-gnome.toml lines 72-78](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L72-L78).
- **Keep the BAKE Flatpak source of truth.**
  Rationale: `installer/flatpaks-base` is the current list, and the existing
  BIB ISO rsyncs the baked repo into the target:
  [flatpaks-base lines 1-61](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/installer/flatpaks-base#L1-L61) and
  [iso-gnome.toml lines 220-278](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L220-L278).
- **Keep the partitioning contract unless the project lead accepts a UX
  change.**
  Rationale: the existing ISO config chooses disks and creates the current
  ESP/Btrfs layout:
  [iso-gnome.toml lines 34-70](https://github.com/daniel-g-carrasco/margine-image/blob/36f0234f5b1d1bc5bdfac35f209d8026945b17e9/disk_config/iso-gnome.toml#L34-L70).
- **Keep a BIB fallback until one Titanoboa ISO has passed hardware Secure
  Boot testing.**
  Rationale: Bluefin's current public ISO workflows show failures in recent
  API metadata, and Bazzite has not shown a public post-#138 green run in the
  latest runs inspected. This is a risk-control rule, not a claim that
  Titanoboa cannot work.

## 4. Margine-Specific Blockers And Resolutions

| Blocker | Why it matters | Proposed resolution | Effort |
| --- | --- | --- | --- |
| Current Titanoboa cannot consume `disk_config/iso-gnome.toml`. | #138 removed workflow-level customization inputs, and `build_iso.sh` only reads `iso.yaml` from the image. | Add a Margine live-environment image that writes `/usr/lib/bootc-image-builder/iso.yaml`, Anaconda profile/defaults, post scripts, Flatpaks, and install payload. | M |
| Secure Boot live ISO chicken-and-egg. | A live ISO using only a Margine-MOK-signed CachyOS kernel will not boot under Secure Boot until the MOK is already enrolled. | Use Bazzite's pattern: keep the installed payload on Margine/CachyOS, but use a Fedora-signed live ISO kernel/initramfs path for the installer environment. | L |
| Existing MOK import lives in the BIB Kickstart file. | The project lead reported the prompt failure after first installed reboot; this is the exact flow the migration must preserve and improve. | Move the MOK import logic into the generated Anaconda `%post --nochroot` script in the live image, preserving the PR #88 behavior and requiring a hardware Secure Boot test. | S/M |
| BAKE Flatpaks are currently produced by a transient BIB installer image. | Install-time Flatpak work has already failed silently once, and the current fix relies on build-time baking plus rsync. | Bake Flatpaks into the live-environment image, mount/preserve `/var/lib/flatpak`, then rsync into the target in Anaconda post script. Bazzite's read-only bind pattern is the reference. | M |
| Existing partitioning is in a BIB config file. | The Anaconda UX and partition layout are part of the product behavior. | Generate `interactive-defaults.ks` or included Kickstart fragments from Margine scripts and port the disk-selection/partition commands from `iso-gnome.toml`. | M |
| `bootc switch` is currently in BIB `%post`. | The installed system must track `ghcr.io/daniel-g-carrasco/margine:stable`, not the transient installer/live image. | Keep the same switch command in Anaconda post script. | S |
| Bluefin's current workflow is stale against Titanoboa #138. | Copying it would carry old inputs that current Titanoboa ignores or rejects. | Use Bazzite current source and Titanoboa current examples for workflow shape; use Bluefin only for Anaconda/MOK script ideas. | S |
| ISO size and runner disk pressure may change but not disappear. | Titanoboa still builds a SquashFS and embedded payload; the ISO may remain large. | Measure the first prototype against the 9 GB project-lead baseline. Keep GitHub free-space cleanup and add Bazzite-style Btrfs loopback storage only if needed. | M |
| Public proof is incomplete after the #138 rewrite. | Bluefin recent API metadata shows failures, and Bazzite latest inspected public run predates #138. | Prototype locally and in CI before deleting BIB. Keep the fallback until a real hardware Secure Boot install succeeds. | M |
| Offline docs tarball hosting may move. | The current build system has separate artifact concerns that could change with a new ISO workflow. | Treat docs tarball hosting as an open decision for the implementation PR, not this ADR. | S |

## 5. Migration Phases - Week-By-Week Plan

Target range: 10-12 working days, with a hard 14-day cap before reassessment.

1. **Live-environment scaffold**

   **Description:** Create an `installer-live/` or `iso/` directory in
   `margine-image` with:

   - `Containerfile`
   - `build.sh`
   - `/usr/lib/bootc-image-builder/iso.yaml`
   - pre-initramfs and post-rootfs scripts
   - copied or generated Anaconda profile/defaults
   - copied BAKE Flatpak list from `installer/flatpaks-base`

   **Acceptance criteria:**

   - `podman build` succeeds locally or in CI.
   - The built image contains `iso.yaml`, `vmlinuz`, `initramfs.img`, EFI
     assets, `anaconda-live`, `dracut-live`, and livesys units.
   - The image does not contain the private MOK key.

   **Estimated duration:** 2 days.

2. **Port the Anaconda install contract**

   **Description:** Translate the current BIB Kickstart behavior into the
   live image:

   - disk detection and partitioning
   - `bootc switch --mutate-in-place --transport registry`
   - Margine MOK import
   - Btrfs tuning
   - BAKE Flatpak rsync
   - Anaconda profile/module defaults

   **Acceptance criteria:**

   - `bash -n`/`shellcheck` pass on generated scripts where practical.
   - `ksvalidator` passes if available in the build environment.
   - A review diff maps each current `iso-gnome.toml` functional block to a
     new live-image script or an explicitly accepted deletion.

   **Estimated duration:** 2 days.

3. **Secure Boot live-kernel strategy**

   **Description:** Implement the live ISO Secure Boot strategy before chasing
   polish. Recommended path: copy Bazzite's model and use a Fedora-signed live
   installer kernel while leaving the installed payload as Margine/CachyOS.
   Bazzite's kernel swap is the reference:
   [Bazzite preinitramfs lines 5-21](https://github.com/ublue-os/bazzite/blob/c8e3f5e90c31c9f06e2e618221b523457b1303eb/installer/titanoboa_hook_preinitramfs.sh#L5-L21).

   **Acceptance criteria:**

   - Live ISO boots in an OVMF Secure Boot VM without pre-enrolled Margine
     MOK.
   - Installed system stages Margine MOK enrollment before its first boot.
   - Project lead hardware test confirms MokManager appears after install.

   **Estimated duration:** 2 days.

4. **Titanoboa workflow prototype**

   **Description:** Add a non-publishing workflow that:

   - builds the Margine live-environment image
   - calls `ublue-os/titanoboa` pinned to a commit, not floating `main`
   - passes only `image-ref` and `iso-dest`
   - uploads ISO and checksum artifacts

   **Acceptance criteria:**

   - ISO artifact is produced.
   - ISO size is measured against the project-lead 9 GB baseline.
   - The BIB `anaconda-iso` path still exists and can be manually dispatched.

   **Estimated duration:** 2 days.

5. **End-to-end install tests**

   **Description:** Run install tests in this order:

   - no-Secure-Boot VM install
   - Secure Boot VM install with MOK enrollment
   - project-lead hardware install

   **Acceptance criteria:**

   - Installed system boots into Margine.
   - `bootc status` points to the Margine stable image.
   - Baked Flatpaks are present without network-time install work.
   - Partition layout matches the current contract or an explicitly approved
     replacement.
   - Secure Boot and MOK behavior matches the documented first-boot flow.

   **Estimated duration:** 2 days.

6. **Cutover and fallback**

   **Description:** Make Titanoboa the default ISO path only after Phase 5
   passes.

   **Acceptance criteria:**

   - Release artifact publication uses the Titanoboa ISO.
   - BIB remains available as a manual fallback for one or two releases.
   - Docs and validator expectations are updated.
   - A rollback procedure exists: rebuild the last BIB ISO path and republish
     its checksum.

   **Estimated duration:** 2 days.

## 6. Open Decisions

- **Live image shape:** separate `margine-live-env`/`margine-installer` image
  versus extending the main Margine image. Recommendation: separate live image,
  because it allows Fedora-signed live-kernel behavior and installer packages
  without polluting the installed OS.
- **Titanoboa pin:** `ublue-os/titanoboa@5c457c3d0518bd17e754be0fd98a60d29d26abb4`
  versus a newer commit/tag when implementation starts. Recommendation: pin a
  commit for the first prototype, then update intentionally.
- **Installer future:** Anaconda versus Readymade/Calamares later.
  Recommendation: Anaconda for phase 1; revisit only after the Titanoboa ISO
  boots and installs reliably.
- **Live ISO Secure Boot policy:** Fedora-signed live kernel versus requiring
  users to disable Secure Boot for the installer. Recommendation: Fedora-signed
  live kernel; requiring Secure Boot disablement would be a regression from
  Bluefin behavior reported by the project lead.
- **Fallback lifetime:** one release or two releases with BIB still available.
  Recommendation: two successful Titanoboa ISO releases before removing BIB.
- **Artifact publishing:** keep the existing release/Internet Archive layout or
  create a Titanoboa-specific artifact path. Recommendation: keep filenames and
  publication semantics stable unless ISO size or checksum generation forces a
  change.
- **Offline docs tarball:** keep current artifact flow or bake docs into the
  live image. Recommendation: decide during implementation after measuring ISO
  size impact.
- **ISO label and boot menu:** `Margine-Live` and a minimal fallback
  `nomodeset` entry are likely, but this needs project-lead confirmation before
  publishing.

## Consequences

Positive:

- Margine aligns with Titanoboa's current upstream direction instead of the
  deprecated pre-#138 input model.
- The migration keeps Anaconda, BAKE Flatpaks, Secure Boot/MOK enrollment, and
  the stable bootc origin as first-class requirements.
- The live ISO can be tested independently from BIB, with a fallback preserved.

Negative:

- The migration is not a simple workflow swap. Most of the current ISO config
  has to move into a purpose-built live image.
- Secure Boot needs explicit live-kernel work. Without it, Margine risks an
  installer that only works when Secure Boot is disabled.
- The first Titanoboa ISO may still be large. Titanoboa changes the build
  mechanism; it does not automatically remove the embedded payload/Flatpak
  size.

Risk controls:

- Keep BIB manual fallback through at least one successful hardware-tested
  Titanoboa release.
- Pin Titanoboa by commit while the project is moving quickly.
- Treat Bluefin as an Anaconda script reference, not a current workflow
  reference, until its workflow matches the post-#138 action API.

## 7. References

- Titanoboa current tree:
  <https://github.com/ublue-os/titanoboa/tree/5c457c3d0518bd17e754be0fd98a60d29d26abb4>
- Titanoboa README:
  <https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/README.md>
- Titanoboa action:
  <https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/action.yml>
- Titanoboa runner scripts:
  <https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/main.sh> and
  <https://github.com/ublue-os/titanoboa/blob/5c457c3d0518bd17e754be0fd98a60d29d26abb4/build_iso.sh>
- Titanoboa breaking PRs:
  <https://github.com/ublue-os/titanoboa/pull/138> and
  <https://github.com/ublue-os/titanoboa/pull/38>
- Titanoboa dependents:
  <https://github.com/ublue-os/titanoboa/network/dependents>
- Bluefin ISO repo:
  <https://github.com/projectbluefin/iso/tree/a4e89e2a93009035621ccd2713c9ff9595393857>
- Bazzite repo:
  <https://github.com/ublue-os/bazzite/tree/c8e3f5e90c31c9f06e2e618221b523457b1303eb>
- Aurora repo:
  <https://github.com/ublue-os/aurora/tree/90cd49c575780fde28efb56acdc3940b2d34095f>
- Cosmic repo:
  <https://github.com/ublue-os/cosmic/tree/c498ba6720065a4bf76e8c49b087557f58ea1cff>
- pykickstart Kickstart docs:
  <https://pykickstart.readthedocs.io/en/latest/kickstart-docs.html>
- Fedora package metadata for live ISO packages:
  <https://packages.fedoraproject.org/pkgs/dracut/>,
  <https://packages.fedoraproject.org/pkgs/dracut/dracut-live/fedora-44.html>,
  <https://packages.fedoraproject.org/pkgs/livesys-scripts/livesys-scripts/>,
  <https://packages.fedoraproject.org/pkgs/grub2/grub2-efi-x64-cdboot/>, and
  <https://packages.fedoraproject.org/pkgs/anaconda/anaconda-live/>
- Red Hat Secure Boot and MOK docs:
  <https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_monitoring_and_updating_the_kernel/index>
- Calamares:
  <https://github.com/calamares/calamares>
- Readymade docs and repo:
  <https://developer.fyralabs.com/rdms> and
  <https://github.com/FyraLabs/readymade>
