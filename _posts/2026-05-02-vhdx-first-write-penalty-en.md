---
title: "VHDX-Dynamic's 32 MiB block default is a cloud storage kernel trap"
date: 2026-05-02 18:00:00 +0900
categories: [Windows Analyze, FileSystem]
tags: [vhdmp, vhdx, kernel, hyper-v, block-storage]
lang: en
---

🇰🇷 [한국어 버전](/posts/vhdx-first-write-penalty-ko/) ・ 🇺🇸 English (you are here)

> **TL;DR**
> - Of 5 virtual disk formats, only VHDX-Dynamic is 15× slower on first writes (21 vs 318 IOPS).
> - Root cause is not the format but `BlockSizeInBytes` default (32 MiB) — VHDX-Differencing (2 MiB, same engine) matches VHD-Dynamic.
> - For host/kernel-side prefill of fresh disks, override with `BlockSizeInBytes = 2 * 1024 * 1024`.
{: .prompt-tip }

Write kernel-side code behind a Hyper-V class hypervisor — virtual disk
drivers, cloud storage allocators, snapshot engines — and you eventually
meet `vhdmp.sys` and the VHDX format. Internal docs treat the choice
between VHD-Fixed, VHD-Dynamic, VHDX-Fixed, VHDX-Dynamic, and
VHDX-Differencing as a meaningful performance decision. End-to-end
measurement says otherwise: after warm-up the five formats land within
5% of each other on every workload.

The exception is the first write. On a 4 KiB sequential write that
touches each block for the first time, VHDX-Dynamic ran at **21 IOPS**
versus VHD-Dynamic's **318 IOPS** — a 15× slowdown for the format that
was supposed to be the upgrade.

| Format             | first-write IOPS | p50    | p99      | max       |
| ------------------ | ---------------: | -----: | -------: | --------: |
| VHD-Fixed          | (pre-allocated)  | —      | —        | —         |
| VHD-Dynamic        |            318.2 |  124us |   67.9ms |   150.8ms |
| VHDX-Fixed         | (pre-allocated)  | —      | —        | —         |
| **VHDX-Dynamic**   |         **21.1** |  283us |  151.7ms |   210.7ms |
| VHDX-Differencing  |            317.1 |  1.2ms |   12.5ms |    54.2ms |

The format isn't broken. The only meaningful difference between these
two formats on this workload is one default value almost no one
overrides.

## The default that no one sets

Call `CreateVirtualDisk` for a VHDX-Dynamic and the API picks
`BlockSizeInBytes = 32 MiB`. For VHD-Dynamic it picks 2 MiB. Right
there in the wrapper:

```cpp
ULONGLONG DefaultBlockSize(DiskFormat f) {
    switch (f) {
    case DiskFormat::VhdDynamic:       return 2ULL  * 1024 * 1024;  //  2 MiB
    case DiskFormat::VhdxDynamic:      return 32ULL * 1024 * 1024;  // 32 MiB ← 16× larger
    case DiskFormat::VhdxDifferencing: return 2ULL  * 1024 * 1024;  //  2 MiB
    default:                            return 0;                   // Fixed: N/A
    }
}
```
{: file='vdisk_factory.cpp' }

The first-write phase issues 4 KiB writes to every block's first byte,
then second byte, across a 2 GiB IO range. VHD-Dynamic allocates ~1024
blocks of 2 MiB. VHDX-Dynamic allocates 64 blocks of 32 MiB — and zeros
32 MiB of disk per block before the 4 KiB write can land. The 16× larger
zero-fill, multiplied by VHDX's slightly heavier metadata path, lands at
the 15× number you measured.

The actual call:

```cpp
CREATE_VIRTUAL_DISK_PARAMETERS p = {};
p.Version = CREATE_VIRTUAL_DISK_VERSION_2;
p.Version2.MaximumSize           = sizeBytes;
p.Version2.BlockSizeInBytes      = 0;     // ← 0 means "use default"
p.Version2.SectorSizeInBytes     = 512;
p.Version2.PhysicalSectorSizeInBytes = isVhdx ? 4096 : 0;

CREATE_VIRTUAL_DISK_FLAG flags = CREATE_VIRTUAL_DISK_FLAG_NONE;
if (format == DiskFormat::VhdFixed || format == DiskFormat::VhdxFixed) {
    flags = CREATE_VIRTUAL_DISK_FLAG_FULL_PHYSICAL_ALLOCATION;
}

CreateVirtualDisk(&st, path, VIRTUAL_DISK_ACCESS_NONE,
                  nullptr, flags, 0, &p, nullptr, &handle);
```
{: file='vdisk_factory.cpp' }

`BlockSizeInBytes = 0` is the trap. Code copy-pasted from the MSDN sample
leaves it that way and the OS substitutes the format default. Fine for
VHD-Dynamic, catastrophic for VHDX-Dynamic on this workload.

## The smoking gun

The third format — VHDX-Differencing — gives it away. Same VHDX engine,
same kernel driver path (`vhdmp.sys`), same metadata layout. Default
block size 2 MiB. First-write number 317 IOPS — essentially identical to
VHD-Dynamic.

The variable isn't the format. It's the block size.

## So is the default wrong?

No. VHDX-Dynamic's 32 MiB default targets guest VM workloads, where
allocations happen during OS install or first boot, amortize over months
of VM lifetime, and where smaller blocks would mean a larger BAT and
more metadata churn during normal operation. For a guest VM, it's the
right call.

For a host-side script — or anyone in the kernel storage path
benchmarking, prefilling, or copying data into newly-allocated cloud
disks — it's the wrong call. That workload was never the target.

The override is one line:

```cpp
p.Version2.BlockSizeInBytes = 2 * 1024 * 1024;  // force 2 MiB explicitly
```

With `BlockSizeInBytes = 2 MiB`, VHDX-Dynamic's first-write throughput
matches VHD-Dynamic to within a few percent. You keep what actually
matters about VHDX (>2 TB capacity, 4 KiB sector alignment, log-based
corruption resistance) and stop paying for a default tuned for a
different workload.

## Why I think this matters at the kernel/cloud layer

- **Provisioning fleets**: at cloud scale you create thousands of
  VHDX-backed disks. If the provisioning kernel path uses the default
  block size and immediately writes scratch data into the new disk
  (signature, MBR, fs metadata), you pay this penalty once per disk —
  fleet size as the multiplier.
- **Backup restore**: restoring a VHDX from snapshot allocates blocks on
  first write. If the restore path streams data at 4 KiB granularity,
  the penalty hits the kernel layer with no warning.
- **VHDX-Differencing as snapshot fabric**: this looks like the closest
  primitive to a cloud snapshot's copy-on-write behavior. Differencing
  first-writing at 317 IOPS instead of 21 is the reason I'd reach for it
  over VHDX-Dynamic in clone/snapshot scenarios where the parent
  already exists.
- **Steady state is fine**: once a disk is fully written through, all
  five formats sit within 5% of each other on every per-IO metric.

## Steady-state confirmation

| Workload (4K rand R QD32) | IOPS  |
| ------------------------- | ----: |
| VHD-Fixed                 | 84.8K |
| VHD-Dynamic               | 84.2K |
| VHDX-Fixed                | 84.3K |
| VHDX-Dynamic              | 82.2K |
| VHDX-Differencing         | 84.3K |

All five within 4%. After warm-up, format affects on-disk shape, not
runtime cost.

## The one thing to remember

**The only performance number that distinguishes the five virtual disk
formats is the first-write penalty, and it is entirely a function of
`BlockSizeInBytes` — not the format itself.** Once that lands,
"should we use VHD or VHDX?" stops being a performance question and
becomes a feature question.

---

## Attached data

- 📥 [aggregate.csv (22 KB)](/assets/data/vhdx-2026-05-02/aggregate.csv){:download="aggregate.csv"} — per-cell mean / stddev across all 5 iterations
- 📥 [report.txt (22 KB)](/assets/data/vhdx-2026-05-02/report.txt){:download="report.txt"} — human-readable comprehensive report
- 📥 [virtualdisk_etw.wprp (1.6 KB)](/assets/data/vhdx-2026-05-02/virtualdisk_etw.wprp){:download="virtualdisk_etw.wprp"} — ETW capture profile

Related:
- [Hyper-V vhdmp.sys baseline: 5 formats × 158 workloads (dataset)](/posts/virtual-disk-benchmark-data-en/)
- [Tracing vhdmp.sys: why your WPR session captures zero VHDMP events](/posts/vhdmp-etw-zero-events-en/)
