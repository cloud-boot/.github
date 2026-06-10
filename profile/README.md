# cloud-boot

**Boot unmodified cloud images on any UEFI hypervisor.**

`cloud-boot` is a family of repos that together let you take a stock
distro cloud image and boot it — without rebuilding the image, without
trusting yet another bootloader, without per-hypervisor glue — on
KVM/QEMU/OVMF, Apple `Virtualization.framework`, OpenStack, and bare
metal. Pure Go from PID 1 down to the firmware-facing PE32+ entry
point; no cgo, no kernel modules required.

Two complementary tracks coexist:

- **Phase 1 — UKI toolchain + Linux-side init.** Three boot paths
  (`init` + `kexec` ; UKI menu-then-reboot ; pure-UEFI TinyGo loader)
  that all land on the same end state — a stock distro userspace from
  an unmodified cloud disk. Covers six Linux families and three BSDs
  on four architectures.
- **Phase 2 — pure-Go bare-metal UEFI loader on TamaGo.** A PXE-class
  pre-boot agent that runs inside UEFI Boot Services, walks PCI,
  drives virtio-net, speaks DHCPv4 / DNS / TLS / HTTPS / OCI
  Distribution v2, verifies cosign signatures, then `LoadImage` +
  `StartImage`s the distro kernel, lands in a stock Debian
  userspace. **Live end-to-end Linux userspace on all four
  arches** (amd64 + arm64 + riscv64 + loong64) as of 2026-06-10 ;
  closed via the `R-amd64a..j` cleanup chain (EDK2 CpuPageTableLib
  upstream fix, TamaGo cpuinit `.bss`/`goos.Bloc` writes, dialTLS
  deadline-math, and the OVMF amd64 `LoadFile2` quirk worked around
  via the `initrd=` kernel cmdline path).

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
  -> LoadImage -> SetLoadOptions(cmdline) -> PublishDTB -> PublishInitrd
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
lease. The amd64 leg uses a different initrd-handoff mechanism than
the other three — EDK2 OVMF amd64 silently mis-handles the
`LoadFile2` protocol the kernel expects, so the loader falls back to
the long-standing `initrd=` kernel-cmdline path published as a
`SimpleFileSystem` ESP file ; the kernel reaches the same end state
either way. Closure landed via the `R-amd64a..j` chain (EDK2
`CpuPageTableLib` upstream fix, TamaGo cpuinit `.bss` zeroing +
`goos.Bloc` MOVQ, `rxLoop` allocation churn fix, `dialTLSOnce`
deadline-math fix, then `R-amd64j` espfile workaround).

Sharp edges still tracked : `R-M9.1a` virtio-console firmware
handoff, `R-M9.2a` `time.Sleep` under TamaGo+UEFI.

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
