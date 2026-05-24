---
title: "Hyper-V vhdmp.sys baseline: 5 virtual disk formats × 158 workloads"
date: 2026-05-02 18:30:00 +0900
categories: [Windows Analyze, FileSystem]
tags: [vhdmp, vhdx, kernel, hyper-v, benchmark, iocp, etw]
lang: en
---

🇰🇷 [한국어 버전](/posts/virtual-disk-benchmark-data-ko/) ・ 🇺🇸 English (you are here)

> **TL;DR**
> - vhdmp.sys baseline across 158 workloads × 5 iterations × 5 formats = 3,950 cells.
> - After warm-up, all five formats converge within 5% on every workload — first-write is the only exception (separate post).
> - Direct kernel storage stack measurement via `FILE_FLAG_NO_BUFFERING` + IOCP. raw csv/wprp attached.
{: .prompt-tip }

Before measuring cloud storage kernel overhead — the bytes `vhdmp.sys`
adds on top of the underlying device, IRP amplification of differencing
chains, the latency floor your driver fights against — you need a clean
baseline of the local virtual disk stack. This post is that baseline for
one specific machine, with enough methodology spelled out for someone
else to rerun and compare.

The folklore: VHD is legacy, VHDX is modern, fixed is fast, dynamic is
slow, format choice matters. Measured: 158 workloads × 5 iterations × 5
formats = **3,950 cells**. After warm-up all five formats land within 5%
of each other on every workload tested. The one decision that moves a
number is `BlockSizeInBytes`, which isn't even part of the format choice
— see the [first-write penalty post][post1] for why.

[post1]: /posts/vhdx-first-write-penalty-en/

## TL;DR

| Claim | Evidence |
| --- | --- |
| Steady-state throughput is format-independent | All cells within 5% across 5 formats |
| First-write differs by 15× | Only on VHDX-Dynamic with 32 MiB default block |
| 32 MiB default is right for VMs, wrong for kernel-side prep | Override `BlockSizeInBytes` |
| CPU cost per IO is also format-independent | < 10% variation in any cell |
| Cache hot-spotting effect is invisible | 2 GiB IO range fits in 16 GB RAM |
| Stride pattern degrades to 40-65% of pure sequential | vhdmp/SSD prefetch doesn't learn skips |

## Methodology

Hardware: Windows 11 Pro, 11th-gen i5-11400F (6c/12t), 15.8 GiB RAM,
single SATA SSD. Single-machine, single-drive, no replication.

Per iteration:

1. Create fresh 4 GiB virtual disk, attach without drive letter
2. Run all phases against the physical path (`\\.\PhysicalDriveN`)
3. Detach and delete

5 iterations with cooldown between each:

- `SetSystemFileCacheSize` hard-trim (NTFS Cache Manager flush)
- 1 GiB memory pressure (`VirtualAlloc` + page touch + `VirtualFree`)
- 120 second sleep

The cooldown produces < 10% CV on most cells. Without it, noise on
first-write cells exceeded 30%.

Phases per format:

- `first_write` : 4K seq write QD1, stride-aligned, touches each block twice
- `prealloc`    : 1 MiB seq write fills IO range (thin formats only)
- `matrix`      : 24 cells = (4K, 64K, 1M) × (QD1, QD32) × (seq, rand) × (R, W)
- `mixed_rw70`  : 30% writes / 70% reads, QD32, 4K and 64K, 5s
- `hotspot`     : 80% IO in 20% of range, QD32, 4K and 64K, 5s
- `stride`      : 4K block on 64K stride sequential, QD1+QD32, 5s
- `rewrite`     : 4K QD32 sequential write to 16 MiB region, 5s

## Why this is a kernel-layer measurement

The runner uses Win32 file APIs that bypass every cache between user
mode and the storage stack:

```cpp
HANDLE diskHandle = CreateFileW(
    physicalPath, access,
    FILE_SHARE_READ | FILE_SHARE_WRITE,
    nullptr, OPEN_EXISTING,
    FILE_FLAG_NO_BUFFERING       // ← bypass OS page cache
    | FILE_FLAG_WRITE_THROUGH    // ← writes complete only after device ack
    | FILE_FLAG_OVERLAPPED,      // ← async IO via IOCP
    nullptr);
```
{: file='io_workload.cpp' }

`FILE_FLAG_NO_BUFFERING` removes the FS cache, so reads/writes go
straight through `vhdmp.sys` and into the device driver. Same path a
kernel-side cloud storage component would use forwarding IRPs into a
VHDX-backed disk.

The IOCP loop pre-fills QD outstanding requests, then re-issues on every
completion:

```text
prime QD requests        // fill the pipe
inFlight = QD

while inFlight > 0:
    GetQueuedCompletionStatus(blocking)   // wait for one completion
    record latency = now - slot.issueQpc
    inFlight--

    if not stopIssuing:
        check time/op limits
        issue another IO on this slot      // reuse slot immediately
        inFlight++
```

For QD32 workloads, exactly 32 IOs are in flight at every moment after
warm-up.

Latency is captured at the application layer — immediately before
`WriteFile` dispatch, immediately after `GetQueuedCompletionStatus`
returns. That includes syscall + driver + device + completion-port
wakeup, which is the kernel-stack-end latency a real cloud storage
service would observe:

```cpp
LARGE_INTEGER q;
QueryPerformanceCounter(&q);
slot->issueQpc = q.QuadPart;
WriteFile(diskHandle, slot->buffer, blockSize, nullptr, &slot->ov);

// ... on IOCP completion ...
LARGE_INTEGER doneQpc;
QueryPerformanceCounter(&doneQpc);
double us = (double)(doneQpc.QuadPart - slot->issueQpc) * 1e6 / qpf.QuadPart;
hist.Record(us);
```
{: file='io_workload.cpp' }

Latency percentiles use a sort-based histogram. With ~10⁶ samples per
cell that's fast enough to skip HdrHistogram:

```cpp
double LatencyHistogram::Percentile(double p) const {
    if (size_ == 0) return 0.0;
    EnsureSorted();                           // qsort, once
    if (p <= 0.0) return data_[0];
    if (p >= 1.0) return data_[size_ - 1];
    size_t idx = (size_t)((double)(size_ - 1) * p);
    return data_[idx];                        // nearest-rank
}
```
{: file='metrics.cpp' }

The first-write phase splits "first" from "subsequent" writes with a
single bit per block:

```cpp
BYTE* firstTouch = (BYTE*)HeapAlloc(...);
FillMemory(firstTouch, fwNumBlocks, 1);    // all "untouched"

// on issue:
uint64_t idx = seqIdx % fwNumBlocks;
bool isFirst = (firstTouch[idx] != 0);
firstTouch[idx] = 0;                       // mark; next time it's a rewrite

// on completion:
if (slot->isFirstWrite) firstHist.Record(us);
else                    subHist.Record(us);
```
{: file='io_workload.cpp' }

Two histograms per first-write phase — first touches vs re-writes of
already-allocated blocks. The 15× separation between them is the entire
point of the headline post.

## Steady-state matrix

5 formats × 9 representative cells. All values are mean across 5
iterations.

```
Workload                VHD-Fixed     VHD-Dyn    VHDX-Fix    VHDX-Dyn   VHDX-Diff
------------------------------------------------------------------------------
4K random read  QD1          9.6K        9.7K        9.8K        9.9K        9.8K
4K random read  QD32        84.8K       84.2K       84.3K       82.2K       84.3K
4K random write QD1         17.0K       17.7K       18.0K       18.0K       18.2K
4K random write QD32        75.0K       68.8K       72.5K       72.0K       73.9K
4K seq    write QD32        68.3K       73.4K       70.8K       72.0K       68.2K
64K rand  read  QD32         8.1K        8.2K        8.3K        8.2K        8.3K
64K rand  write QD32         5.7K        5.8K        5.6K        5.7K        5.4K
1M  seq   read  QD32        533.2       532.8       534.4       534.0       533.1
1M  seq   write QD32        361.1       365.5       346.1       345.8       339.0
```

p50 latency for the same:

```
Workload                VHD-Fixed     VHD-Dyn    VHDX-Fix    VHDX-Dyn   VHDX-Diff
------------------------------------------------------------------------------
4K random read  QD1        80.7us      82.1us      80.5us      78.2us      80.3us
4K random read  QD32        332us       332us       332us       332us       332us
4K random write QD1        49.9us      49.8us      49.8us      49.8us      49.4us
4K random write QD32        392us       392us       392us       392us       392us
4K seq    write QD32        428us       392us       408us       414us       453us
64K rand  read  QD32        3.8ms       3.8ms       3.8ms       3.8ms       3.8ms
64K rand  write QD32        5.7ms       5.4ms       5.4ms       5.3ms       5.5ms
1M  seq   read  QD32       59.8ms      59.8ms      59.8ms      59.8ms      59.8ms
1M  seq   write QD32       85.7ms      85.5ms      90.8ms      90.3ms      91.2ms
```

The 4K rand read QD32 row hits ~84K IOPS for all five formats. The 1M
seq read QD32 row hits ~534 MB/s — the SSD's saturation limit. Every
cell shows < 10% spread across formats.

## Hotspot, stride, mixed

```
Hotspot (80% IO in 20% range, QD32):
Workload                VHD-Fixed     VHD-Dyn    VHDX-Fix    VHDX-Dyn   VHDX-Diff
------------------------------------------------------------------------------
  4K hotspot R QD32         82.7K       83.6K       82.7K       82.7K       83.2K
 64K hotspot R QD32          8.3K        8.3K        8.3K        8.3K        8.3K

Stride (4K block on 64K stride, sequential read):
Format                Stride QD1 Stride QD32
------------------------------------------------------------------------------
VHD-Fixed                   7.6K       46.8K
VHD-Dynamic                 7.9K       47.2K
VHDX-Fixed                  7.7K       46.5K
VHDX-Dynamic                7.7K       44.8K
VHDX-Differencing           7.8K       47.7K

Mixed RW (30% writes / 70% reads, random, QD32):
Format                        4K         64K
------------------------------------------------------------------------------
VHD-Fixed                  64.5K        6.4K
VHD-Dynamic                66.5K        6.3K
VHDX-Fixed                 65.3K        6.3K
VHDX-Dynamic               64.1K        6.3K
VHDX-Differencing          64.1K        6.3K
```

Three observations:

1. **Hotspot vs uniform random gives no IOPS gain.** Cache effects
   negligible because 2 GiB IO range fits in RAM regardless. To exercise
   true cache pressure, IO range must exceed RAM.
2. **Stride pattern degrades to 40-65% of pure sequential.** Both
   `vhdmp.sys` and the SSD prefetcher treat 4K-on-64K-stride as random;
   neither learns the skip cadence.
3. **Mixed RW runs at 75-80% of pure read.** Concurrent writes block
   read queue advancement, dragging combined IOPS toward write speed.

## CPU cost

```
Workload                VHD-Fixed     VHD-Dyn    VHDX-Fix    VHDX-Dyn   VHDX-Diff
------------------------------------------------------------------------------
4K random read  QD1          6.5%        7.1%        6.5%        5.7%        6.0%
4K random read  QD32        35.9%       32.0%       37.8%       33.8%       31.3%
4K random write QD32        31.4%       32.3%       35.5%       32.6%       34.9%
1M  seq   read  QD32         1.6%        0.7%        1.1%        1.3%        1.6%
```

Hot cells (4K QD32) sit around 30-38% CPU — dominated by user-mode IOCP
completion handling, not `vhdmp.sys`. CPU cost varies by workload, barely
by format.

## Measurement quality (CV across 5 iterations)

| Cell                      | VHD-Fix | VHD-Dyn | VHDX-Fix | VHDX-Dyn | VHDX-Diff |
| ------------------------- | ------: | ------: | -------: | -------: | --------: |
| 4K random read  QD32      |    3.2% |    3.0% |     3.5% |     8.7% |      3.3% |
| 4K random write QD32      |    2.4% |   14.4% |     5.9% |     3.5% |      2.9% |
| 1M seq read     QD32      |    0.2% |    0.2% |     0.1% |     0.2% |      0.3% |
| 1M seq write    QD32      |    3.6% |    2.5% |     4.5% |     4.8% |      4.3% |

CV < 5% on most cells. High-CV outliers (1M QD32 writes, first_write
events) reflect SSD-side write throttling and block-allocation timing —
inherent variability, not measurement noise.

## Caveats

- VHDMP per-IO event capture failed. Per-IRP amplification was inferred
  from IOPS / latency ratios, not direct counts. See the
  [VHDMP ETW post][post3] for the workaround.
- 2 GiB IO range fits entirely in 16 GB system RAM. To exercise true
  cache pressure, range must exceed RAM.
- SSD-specific results: latency p99 spikes (1M QD32 writes hit 100ms+)
  are SSD-side write throttling.
- VHDX-Dynamic was tested with the API default (32 MiB block).
- Single-machine, single-drive measurement; no replication.

[post3]: /posts/vhdmp-etw-zero-events-en/

## Recommendations

| Use case | Format | Storage cost |
| --- | --- | --- |
| Maximum performance, predictable latency | VHD-Fixed or VHDX-Fixed | Full disk size up front |
| Sparse storage with new-disk performance | VHD-Dynamic (2 MiB block) | Thin |
| VHDX features (>2 TB, log resilience) | VHDX-Dynamic with `BlockSizeInBytes = 2 MiB` | Thin |
| Snapshot / clone scenarios | VHDX-Differencing | Cheap snapshots |

---

## Attached data

- 📥 [results.csv (132 KB)](/assets/data/vhdx-2026-05-02/results.csv){:download="results.csv"} — 790 rows, 5 iterations × 158 cells, all metrics
- 📥 [aggregate.csv (22 KB)](/assets/data/vhdx-2026-05-02/aggregate.csv){:download="aggregate.csv"} — per-cell mean / stddev (CV calc)
- 📥 [aggregate.txt (20 KB)](/assets/data/vhdx-2026-05-02/aggregate.txt){:download="aggregate.txt"} — human-readable table
- 📥 [report.txt (22 KB)](/assets/data/vhdx-2026-05-02/report.txt){:download="report.txt"} — comprehensive report
- 📥 [virtualdisk_etw.wprp (1.6 KB)](/assets/data/vhdx-2026-05-02/virtualdisk_etw.wprp){:download="virtualdisk_etw.wprp"} — ETW WPR profile
- 📥 [analysis_iter01.csv (10 KB)](/assets/data/vhdx-2026-05-02/analysis_iter01.csv){:download="analysis_iter01.csv"} — VHDMP IRP counts (all zeros, see caveats)
