<p align="center"><img src="https://raw.githubusercontent.com/cloud-boot/brand/main/social/cloud-boot.png" alt="cloud-boot" width="720"></p>

# cloud-boot

**Boot unmodified cloud images on any UEFI hypervisor.**

`cloud-boot` is a family of repos that together let you take a stock
distro cloud image and boot it — without rebuilding the image, without
trusting yet another bootloader, without per-hypervisor glue — on
KVM/QEMU/OVMF, Apple `Virtualization.framework`, OpenStack, and bare
metal. Pure Go from PID 1 down to the firmware-facing PE32+ entry
point; no cgo, no kernel modules required.

Three complementary tracks coexist:

- **Phase 1 — UKI toolchain + Linux-side init.** Three boot paths
  (`init` + `kexec` ; UKI menu-then-reboot ; pure-UEFI TinyGo loader)
  that all land on the same end state — a stock distro userspace from
  an unmodified cloud disk. Covers six Linux families and three BSDs
  on four architectures.
- **Phase 2 — pure-Go bare-metal UEFI loader on TamaGo (Linux).** A
  PXE-class pre-boot agent that runs inside UEFI Boot Services, walks
  PCI, drives virtio-net, speaks DHCPv4 / DNS / TLS / HTTPS / OCI
  Distribution v2, verifies cosign signatures, then `LoadImage` +
  `StartImage`s the distro kernel, lands in a stock Debian
  userspace. **Live end-to-end Linux userspace on all four
  arches** (amd64 + arm64 + riscv64 + loong64) as of 2026-06-10 ;
  closed via the `R-amd64a..j` cleanup chain (EDK2 CpuPageTableLib
  upstream fix, TamaGo cpuinit `.bss`/`goos.Bloc` writes, dialTLS
  deadline-math, and the OVMF amd64 `LoadFile2` quirk worked around
  via the `initrd=` kernel cmdline path). Subsequently unified
  across all four arches on the `initrd=` path (`M8.15`,
  [`51f7005`](https://github.com/cloud-boot/tamago-uefi/commit/51f7005)),
  and ~1684 LOC of `LoadFile2` publish code excised
  (`M8.16`, [`2868756`](https://github.com/cloud-boot/tamago-uefi/commit/2868756)).
- **Phase 3 — OS-agnostic OCI boot.** Same TamaGo loader, but instead
  of `LoadImage`-ing a Linux kernel directly, it materialises an
  in-memory UFS2 / FAT / NTFS image containing the target OS's own
  `loader.efi` (or `boot.efi` / `bootmgfw.efi`), publishes it through
  pure-Go `EFI_BLOCK_IO_PROTOCOL` + `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL`
  shims, and chain-boots the OS's native first-stage loader. **OpenBSD
  reaches `boot>` live end-to-end** as of 2026-06-11. FreeBSD selects
  our UFS as `currdev` but OOMs at kernel-load (sprint 2E pending).
  NetBSD and Windows scaffolding shipped, gated on ISO size and a
  real NTFS reader respectively.

## Repositories

| Repo | Role |
| --- | --- |
| [`tamago-uefi`](https://github.com/cloud-boot/tamago-uefi) | Phase 2. Pure-Go TamaGo UEFI application — PCI walk, virtio-net, DHCPv4, DNS, TLS (CCADB roots), HTTPS, OCI v2, cosign, LoadImage chain-boot. Live Linux userspace on **all four arches**. |
| [`init`](https://github.com/cloud-boot/init) | Phase 1 Linux PID 1. Two sinks — `kexec` (Path A) and `reboot` after staging `Boot####` via efivarfs (Path C). Disk-mode openers cover ext4 / xfs / btrfs (single + RAID0/1/10/5/6) / ZFS / LUKS1+LUKS2 via the sibling `go-filesystems/*` and `go-fde/*` orgs. |
| [`uki`](https://github.com/cloud-boot/uki) | Host-side toolchain. `cloud-boot build` / `push` / `label`. Cross-compiles init, assembles initramfs, wraps a UKI via `go-coff/pe`, stages a FAT ESP, emits a hybrid GPT + El Torito ISO. |
| [`loader`](https://github.com/cloud-boot/loader) | Phase 1 Path B. TinyGo PE/COFF UEFI app — finds the kernel at runtime on an unmodified ext4 / xfs / btrfs / UFS2 cloud disk, `LoadImage` + `StartImage` straight from inside Boot Services. Six Linux families + FreeBSD + NetBSD verified. |
| [`kernel`](https://github.com/cloud-boot/kernel) | Reproducible Dockerfiles for the minimal bootstrap Linux kernels (`disk` / `cloud` variants per arch). |
| [`iso`](https://github.com/cloud-boot/iso) | Pure-Go multi-arch hybrid iso9660 + GPT assembler + QEMU/EDK2 boot harness. One ISO embeds `BOOT{X64,AA64,RISCV64,LOONGARCH64}.EFI` ; each firmware reads only its own file. |
| [`docs`](https://github.com/cloud-boot/docs) | mkdocs-material site at <https://cloud-boot.github.io/docs/>. |
| [`virtio-net`](https://github.com/cloud-boot/virtio-net) · [`snp`](https://github.com/cloud-boot/snp) | Companion driver code consumed by the Phase 2 loader. |

## Phase 2 — what the pure-Go UEFI loader does today

The full pipeline runs end-to-end inside UEFI Boot Services on the
real Go runtime (GC, goroutines, scheduler) via
[TamaGo](https://github.com/usbarmory/tamago), with `CGO_ENABLED=0`
and no external linker — PE32+ wrapping uses our own
[`go-coff/peln` + `pectl`](https://github.com/go-coff) :

```text
PCI walk -> virtio-net -> DHCPv4 -> DNS -> TLS (CCADB roots) -> HTTPS
  -> OCI Distribution v2 walk -> multi-arch index -> manifest
  -> streaming blob fetch -> SHA-256 verify -> cosign keyed verify
  -> LoadImage -> SetLoadOptions(cmdline) -> PublishSFS(initrd, ESP)
  -> PublishRNG -> StartImage -> Linux EFI-stub
  -> "Booting Linux Kernel..." -> real distro kernel
```

Live status per arch (2026-06-10) :

| arch | M0..M7 (net + OCI) | M8.0 chain-boot | Live Linux userspace | Wall-clock | Kernel | Initrd handoff |
| --- | --- | --- | --- | --- | --- | --- |
| **arm64**    | ✅ | ✅ | ✅ | 17.1 s | Debian 6.12.90 | `LoadFile2` |
| **amd64**    | ✅ | ✅ | ✅ | 16.1 s | Debian 6.12.90 | `initrd=` cmdline + `InheritParentDeviceHandle` |
| **riscv64**  | ✅ | ✅ | ✅ | 18.1 s | Debian 6.12.90 | `LoadFile2` |
| **loong64**  | ✅ | ✅ | ✅ | 17.1 s | Debian 7.0.12 | `LoadFile2` |

All four legs reach a real Debian 13 userspace from a cold DHCP
lease. After `R-amd64j` closure (the amd64-only `initrd=` cmdline
workaround for the OVMF `LoadFile2` quirk), `M8.15`
([`51f7005`](https://github.com/cloud-boot/tamago-uefi/commit/51f7005))
unified all four arches on the `initrd=` path published as a
`SimpleFileSystem` ESP file with `InheritParentDeviceHandle` ;
`M8.16` ([`2868756`](https://github.com/cloud-boot/tamago-uefi/commit/2868756))
then excised ~1684 LOC of dead `LoadFile2` publish code. The legacy
`R-amd64a..j` saga remains in the git log : EDK2 `CpuPageTableLib`
upstream fix, TamaGo cpuinit `.bss` zeroing + `goos.Bloc` MOVQ,
`rxLoop` allocation churn fix, `dialTLSOnce` deadline-math fix.

`M9.0`/`M9.1`/`M9.2` ship an interactive boot menu — DHCP option 67
points at an OCI artifact whose HCL describes the menu, the loader
renders + handles input + dispatches the chosen entry. `R-M9.1a`
virtio-console handoff and `R-M9.2a` `time.Sleep` under TamaGo+UEFI
are documented as working-as-intended ; nothing else is on the
critical path for Linux.

## Phase 3 — OS-agnostic OCI boot (multi-OS, NEW)

Same TamaGo loader, with two new Go-side firmware shims and a pure-Go
FreeBSD-UFS2 read+write driver. Instead of `LoadImage`-ing a Linux
kernel directly, we materialise an in-memory disk image containing
the target OS's own first-stage loader and chain-boot it through
`gBS->LoadImage` with our own published handle :

```text
... OCI fetch (Linux pipeline above) ...
  -> in-memory UFS2 / FAT / NTFS image (go-filesystems/ufs Mkfs)
  -> PublishBlockIO  (Go-side EFI_BLOCK_IO_PROTOCOL)
  -> PublishSFS      (Go-side EFI_SIMPLE_FILE_SYSTEM_PROTOCOL)
  -> LoadImage(devicepath = "ours\EFI\BOOT\BOOTX64.EFI")
  -> StartImage -> OS's native loader.efi / boot.efi / bootmgfw.efi
  -> kernel + userspace
```

Live status per (OS, arch) target as of 2026-06-11 :

| OS | amd64 | arm64 | riscv64 | loong64 | Status | Reference commit |
| --- | :---: | :---: | :---: | :---: | --- | --- |
| **Linux** (Debian 13) | LIVE | LIVE | LIVE | LIVE | userspace, 16-18 s cold-DHCP | [`51f7005`](https://github.com/cloud-boot/tamago-uefi/commit/51f7005) |
| **OpenBSD** | **LIVE** | — | — | — | `boot>` prompt end-to-end | [`d66d338`](https://github.com/cloud-boot/tamago-uefi/commit/d66d338) |
| **FreeBSD** | partial | — | — | — | `loader.efi` selects our UFS as `currdev`, OOM at kernel-load (sprint 2E pending) | [`c37108f`](https://github.com/cloud-boot/tamago-uefi/commit/c37108f) |
| **NetBSD** | scaffolded | — | — | — | probe + EFI + runner ready, gated on ISO download size (sprint 3.x) | [`d66d338`](https://github.com/cloud-boot/tamago-uefi/commit/d66d338) |
| **Windows** | scaffolded | — | — | — | build pipeline + `BOOTX64-WINDOWSBOOT.EFI` ships, gated on a real NTFS reader (sprint 4.0a, multi-month) | [`44140a9`](https://github.com/cloud-boot/tamago-uefi/commit/44140a9) |

Honest gaps :

- **FreeBSD OOM** — `loader.efi` reaches `currdev` selection against
  our published `SimpleFileSystem`, then runs out of usable memory
  before transferring control to the kernel. Sprint 2E will hunt
  the regression.
- **NTFS reader** — Windows scaffolding currently relies on
  pre-built artifacts ; the real path needs a pure-Go NTFS read
  surface in `go-filesystems/ntfs` (multi-month effort).

New infrastructure shipped along the way :

- **`go-filesystems/ufs`** — read+write driver with `Mkfs` and full
  double-indirect support ; cross-validated against three parallel
  UFS2 sources (pure-Go `Mkfs`, real FreeBSD `raw.xz` extraction,
  docker oracle).
- **`EFI_BLOCK_IO_PROTOCOL`** Go-side publish surface
  ([`ff2b56d`](https://github.com/cloud-boot/tamago-uefi/commit/ff2b56d) +
  [`f5367a1`](https://github.com/cloud-boot/tamago-uefi/commit/f5367a1) +
  [`d3eaa38`](https://github.com/cloud-boot/tamago-uefi/commit/d3eaa38)).
- **`EFI_SIMPLE_FILE_SYSTEM_PROTOCOL`** Go-side publish surface
  ([`2831705`](https://github.com/cloud-boot/tamago-uefi/commit/2831705)).
- **LEAQ-direct `.abi0` entry helper** — discovered in sprint 1.2
  while chasing an ABIInternal-wrapper interpose corner case, then
  ported across arm64/riscv64/loong64 RNG trampolines as defensive
  infrastructure (sprint 1.3,
  [`5874f50`](https://github.com/cloud-boot/tamago-uefi/commit/5874f50)).

## Project standards

- **Pure Go.** No cgo, no external linker, no host toolchain past the
  Go compiler itself. PE32+ wrapping done in-tree by
  [`go-coff/peln`](https://github.com/go-coff/peln).
- **BSD-3-Clause** on every source file.
- **Multi-arch.** amd64 / arm64 / riscv64 / loongarch64 across both
  Phase 1 and Phase 2.
- **Spec-traceable.** Phase 2 drivers carry citations back to the
  Virtio 1.1 spec, the UEFI spec, RFC 2131/1035/5246+8446, and the
  OCI Distribution / Image / cosign specs they implement.

## Sister orgs we use or publish

- [`go-virtio`](https://github.com/go-virtio) — pure-Go,
  transport-agnostic virtio drivers (`common` / `net` / `blk`),
  extracted from the Phase 2 loader.
- [`go-compressions`](https://github.com/go-compressions) — pure-Go
  byte-stream transforms (`lzfse`, `blake3`, …). `lzfse` is the body
  compressor for the `go-coff/efipack` PE32+ self-extractor.
- [`go-coff`](https://github.com/go-coff) — PE32+ / EFI tooling :
  [`peln`](https://github.com/go-coff/peln) (parser + linker),
  [`pectl`](https://github.com/go-coff/pectl) (CLI ; `pectl pack`),
  [`efipack`](https://github.com/go-coff/efipack) (self-extracting
  PE32+ compressor — flate + LZFSE, host-side + per-arch stubs),
  [`pec`](https://github.com/go-coff/pec) (Authenticode signing),
  [`stub`](https://github.com/go-coff/stub) (TinyGo UEFI stub).
- [`go-filesystems`](https://github.com/go-filesystems) — pure-Go
  read-only drivers for ext4, xfs, btrfs (single + RAID0/1/10/5/6),
  ZFS (single + mirror + RAID-Z1/2/3).
- [`go-fde`](https://github.com/go-fde) /
  [`go-crypto`](https://github.com/go-crypto) — LUKS1/LUKS2 + the
  pure-Go primitives ZFS native encryption needs (AES-CCM,
  PBKDF2-HMAC-SHA1, HKDF-SHA512).

## Landing page

- Project landing page : <https://cloud-boot.github.io>
- Versioned docs : <https://cloud-boot.github.io/docs/>
