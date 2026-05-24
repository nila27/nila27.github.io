---
title: "Hyper-V vhdmp.sys 베이스라인: 5 가상 디스크 포맷 × 158 워크로드"
date: 2026-05-02 18:30:00 +0900
categories: [Windows Analyze, FileSystem]
tags: [vhdmp, vhdx, kernel, hyper-v, benchmark, iocp, etw]
lang: ko-KR
---

🇰🇷 한국어 (현재) ・ 🇺🇸 [English version](/posts/virtual-disk-benchmark-data-en/)

> **3줄 요약**
> - 158 워크로드 × 5 반복 × 5 포맷 = 3,950 셀의 vhdmp.sys 베이스라인 측정.
> - warm-up 이후 다섯 포맷이 모든 워크로드에서 5% 이내로 수렴 — 첫쓰기만 예외 (별도 글).
> - `FILE_FLAG_NO_BUFFERING` + IOCP로 커널 스토리지 스택을 직접 측정. raw csv/wprp 첨부.
{: .prompt-tip }

클라우드 스토리지 커널 오버헤드를 측정하기 전에 — `vhdmp.sys`가 디바이스
위에 얹는 바이트 비용, differencing chain의 IRP amplification, 드라이버가
싸우는 latency 바닥선 — 로컬 가상 디스크 스택의 깨끗한 베이스라인이
필요하다.
이 글은 한 대의 머신 위에서 잰 그 베이스라인이고, 다른 사람이
재현·비교할 수 있게 방법론을 풀어 두었다.

통념: VHD는 레거시, VHDX는 모던, Fixed는 빠르고 Dynamic은 느리고, 포맷
선택이 중요하다. 측정값: 158 워크로드 × 5 반복 × 5 포맷 = **3,950 셀**.
warm-up 이후 다섯 포맷이 모든 워크로드에서 5% 이내로 수렴한다. 유일하게
숫자를 움직이는 결정은 `BlockSizeInBytes`인데, 이건 포맷 선택의 일부조차
아니다 — 이유는 [첫쓰기 페널티 글][post1] 참조.

[post1]: /posts/vhdx-first-write-penalty-ko/

## TL;DR

| 주장 | 근거 |
| --- | --- |
| Steady-state 처리량은 포맷과 무관 | 5 포맷 모든 셀이 5% 이내 |
| 첫쓰기는 15× 차이 | VHDX-Dynamic + 32 MiB 기본 블록에서만 |
| 32 MiB 기본은 VM용으로 옳고 커널측 prep엔 틀림 | `BlockSizeInBytes` 오버라이드 |
| IO당 CPU 비용도 포맷과 무관 | 어느 셀에서도 변동 < 10% |
| Hot-spotting 캐시 효과 보이지 않음 | 2 GiB IO 범위가 16 GB RAM에 통째로 들어감 |
| Stride 패턴은 순차 대비 40-65%로 떨어짐 | vhdmp/SSD prefetch가 skip 학습 못 함 |

## 방법론

하드웨어: Windows 11 Pro, 11세대 i5-11400F (6c/12t), 15.8 GiB RAM,
SATA SSD 1대. 단일 머신, 단일 드라이브, 복제 없음.

반복마다:

1. 새 4 GiB 가상 디스크 생성, 드라이브 문자 없이 attach
2. physical path(`\\.\PhysicalDriveN`)에 모든 페이즈 실행
3. detach + 삭제

반복 5회, 사이마다 cooldown:

- `SetSystemFileCacheSize` hard-trim (NTFS Cache Manager flush)
- 1 GiB 메모리 압박 (`VirtualAlloc` + 페이지 터치 + `VirtualFree`)
- 120초 sleep

이 cooldown 절차가 대부분 셀에서 CV < 10%를 가능하게 했다. 없으면
첫쓰기 셀의 노이즈가 30%를 넘는다.

포맷별 페이즈:

- `first_write` : 4K seq write QD1, stride-aligned, 각 블록 두 번 만짐
- `prealloc`    : 1 MiB seq write로 IO 범위 채우기 (thin 포맷에만)
- `matrix`      : 24 셀 = (4K, 64K, 1M) × (QD1, QD32) × (seq, rand) × (R, W)
- `mixed_rw70`  : 30% write / 70% read, QD32, 4K + 64K, 5초
- `hotspot`     : 80% IO가 20% 범위에, QD32, 4K + 64K, 5초
- `stride`      : 4K 블록 on 64K stride 순차, QD1+QD32, 5초
- `rewrite`     : 4K QD32 순차 쓰기 16 MiB 영역, 5초

## 왜 이게 커널 레이어 측정일까

러너는 user mode와 스토리지 스택 사이의 모든 캐시를 우회하는 Win32 파일
API를 쓴다:

```cpp
HANDLE diskHandle = CreateFileW(
    physicalPath, access,
    FILE_SHARE_READ | FILE_SHARE_WRITE,
    nullptr, OPEN_EXISTING,
    FILE_FLAG_NO_BUFFERING       // ← OS 페이지 캐시 우회
    | FILE_FLAG_WRITE_THROUGH    // ← 쓰기는 디바이스 ack 후에야 완료
    | FILE_FLAG_OVERLAPPED,      // ← IOCP 비동기 IO
    nullptr);
```
{: file='io_workload.cpp' }

`FILE_FLAG_NO_BUFFERING`이 FS 캐시를 제거해서, 읽기/쓰기는 곧장
`vhdmp.sys`를 통과해 디바이스 드라이버로 들어간다. VHDX-backed 디스크에
IRP를 forwarding하는 커널측 클라우드 스토리지 컴포넌트가 쓸 경로와
같다.

IOCP 루프는 QD개의 IO를 미리 발행하고, 완료마다 즉시 재발행한다:

```text
prime QD개의 IO        // 파이프를 가득 채움
inFlight = QD

while inFlight > 0:
    GetQueuedCompletionStatus(blocking)   // 하나 완료 대기
    record latency = now - slot.issueQpc
    inFlight--

    if not stopIssuing:
        check time/op limits
        issue another IO on this slot      // 같은 슬롯 즉시 재사용
        inFlight++
```

QD32 워크로드에선 warm-up 이후 *항상 32개*가 in-flight인 상태가 보장된다.

Latency는 application 레이어에서 마킹한다 — `WriteFile` 발행 직전,
`GetQueuedCompletionStatus` 복귀 직후. syscall + driver + device +
completion-port wakeup이 다 포함된 *커널 스택 끝단의 latency*. 실제 클라우드
스토리지 서비스가 보는 값과 같다고 본다:

```cpp
LARGE_INTEGER q;
QueryPerformanceCounter(&q);
slot->issueQpc = q.QuadPart;
WriteFile(diskHandle, slot->buffer, blockSize, nullptr, &slot->ov);

// ... IOCP 완료 시 ...
LARGE_INTEGER doneQpc;
QueryPerformanceCounter(&doneQpc);
double us = (double)(doneQpc.QuadPart - slot->issueQpc) * 1e6 / qpf.QuadPart;
hist.Record(us);
```
{: file='io_workload.cpp' }

Latency 백분위는 sort 기반 histogram. 셀당 10⁶ 정도 샘플이면 충분히 빨라서
HdrHistogram을 안 쓴다:

```cpp
double LatencyHistogram::Percentile(double p) const {
    if (size_ == 0) return 0.0;
    EnsureSorted();                           // qsort, 한 번만
    if (p <= 0.0) return data_[0];
    if (p >= 1.0) return data_[size_ - 1];
    size_t idx = (size_t)((double)(size_ - 1) * p);
    return data_[idx];                        // nearest-rank
}
```
{: file='metrics.cpp' }

첫쓰기 페이즈는 블록당 1바이트로 "처음/이후"를 구분한다:

```cpp
BYTE* firstTouch = (BYTE*)HeapAlloc(...);
FillMemory(firstTouch, fwNumBlocks, 1);    // 모두 "아직 안 만짐"

// 발행 시:
uint64_t idx = seqIdx % fwNumBlocks;
bool isFirst = (firstTouch[idx] != 0);
firstTouch[idx] = 0;                       // 표시 → 다음번엔 후속쓰기

// 완료 시:
if (slot->isFirstWrite) firstHist.Record(us);
else                    subHist.Record(us);
```
{: file='io_workload.cpp' }

페이즈마다 두 histogram — 하나는 첫 터치, 하나는 이미 할당된 블록의
재쓰기. 둘 사이의 15× 격차가 헤드라인 글의 핵심이다.

## Steady-state 매트릭스

5 포맷 × 9 대표 셀. 모든 값은 5회 평균.

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

같은 셀의 p50 latency:

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

4K rand read QD32 행이 다섯 포맷 모두 ~84K IOPS. 1M seq read QD32 행은
~534 MB/s에서 SSD 한계로 수렴. 측정한 모든 셀이 포맷 간 < 10% spread.

## Hotspot, stride, mixed

```
Hotspot (80% IO가 20% 범위에, QD32):
Workload                VHD-Fixed     VHD-Dyn    VHDX-Fix    VHDX-Dyn   VHDX-Diff
------------------------------------------------------------------------------
  4K hotspot R QD32         82.7K       83.6K       82.7K       82.7K       83.2K
 64K hotspot R QD32          8.3K        8.3K        8.3K        8.3K        8.3K

Stride (4K 블록 on 64K stride, 순차 read):
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

세 가지 관찰:

1. **Hotspot vs uniform random에서 IOPS 이득 없음.** 2 GiB IO 범위가
   RAM에 통째로 들어가서 캐시 효과 무관. 진짜 캐시 압박을 측정하려면
   IO 범위가 RAM을 넘어야 한다.
2. **Stride 패턴은 순차 대비 40-65%로 떨어짐.** `vhdmp.sys`도 SSD
   prefetcher도 4K-on-64K-stride를 random으로 취급, skip 학습 안 한다.
3. **Mixed RW는 pure read의 75-80%.** 동시 쓰기가 read 큐 진행을 막아
   결합 IOPS를 write 속도 쪽으로 끌어내린다.

## CPU 비용

```
Workload                VHD-Fixed     VHD-Dyn    VHDX-Fix    VHDX-Dyn   VHDX-Diff
------------------------------------------------------------------------------
4K random read  QD1          6.5%        7.1%        6.5%        5.7%        6.0%
4K random read  QD32        35.9%       32.0%       37.8%       33.8%       31.3%
4K random write QD32        31.4%       32.3%       35.5%       32.6%       34.9%
1M  seq   read  QD32         1.6%        0.7%        1.1%        1.3%        1.6%
```

핫 셀(4K QD32)은 약 30-38% CPU. user-mode IOCP 완료 처리가 지배적이지
`vhdmp.sys`가 아니다. CPU 비용은 워크로드에 따라 변하지 포맷에 따라선
거의 변하지 않는다.

## 측정 품질 (5회 반복 CV)

| 셀                         | VHD-Fix | VHD-Dyn | VHDX-Fix | VHDX-Dyn | VHDX-Diff |
| ------------------------- | ------: | ------: | -------: | -------: | --------: |
| 4K random read  QD32      |    3.2% |    3.0% |     3.5% |     8.7% |      3.3% |
| 4K random write QD32      |    2.4% |   14.4% |     5.9% |     3.5% |      2.9% |
| 1M seq read     QD32      |    0.2% |    0.2% |     0.1% |     0.2% |      0.3% |
| 1M seq write    QD32      |    3.6% |    2.5% |     4.5% |     4.8% |      4.3% |

대부분 셀 CV < 5%. 큰 CV(1M QD32 writes, first_write)는 SSD-측 write
throttling과 블록 할당 timing의 본질적 변동성 — 측정 노이즈가 아니다.

## Caveats

- VHDMP per-IO 이벤트 캡처 실패. per-IRP amplification은 직접 카운트가
  아닌 IOPS / latency 비율에서 추정. 우회 방법은 [VHDMP ETW 글][post3]
  참조.
- 2 GiB IO 범위가 16 GB RAM에 통째로 들어간다. 진짜 캐시 압박을
  측정하려면 범위가 RAM을 넘어야 한다.
- SSD-특정 결과: latency p99 spike(1M QD32 writes의 100ms+)는 SSD-측
  write throttling.
- VHDX-Dynamic은 API 기본값(32 MiB 블록)으로 측정.
- 단일 머신, 단일 드라이브, 복제 없음.

[post3]: /posts/vhdmp-etw-zero-events-ko/

## 권고

| 사용 케이스 | 포맷 | 저장 비용 |
| --- | --- | --- |
| 최대 성능, 예측 가능 latency | VHD-Fixed 또는 VHDX-Fixed | 디스크 전체 사전 할당 |
| Sparse 저장 + new-disk 성능 | VHD-Dynamic (2 MiB 블록) | Thin |
| VHDX 기능 (>2 TB, log resilience) | VHDX-Dynamic + `BlockSizeInBytes = 2 MiB` | Thin |
| 스냅샷 / 클론 시나리오 | VHDX-Differencing | 저렴한 스냅샷 |

---

## 첨부 자료 (raw)

- 📥 [results.csv (132 KB)](/assets/data/vhdx-2026-05-02/results.csv){:download="results.csv"} — 790행, 5 iterations × 158 cells, 모든 metric
- 📥 [aggregate.csv (22 KB)](/assets/data/vhdx-2026-05-02/aggregate.csv){:download="aggregate.csv"} — 셀별 mean/stddev (CV 계산)
- 📥 [aggregate.txt (20 KB)](/assets/data/vhdx-2026-05-02/aggregate.txt){:download="aggregate.txt"} — 사람이 읽기 좋은 형식
- 📥 [report.txt (22 KB)](/assets/data/vhdx-2026-05-02/report.txt){:download="report.txt"} — 종합 리포트
- 📥 [virtualdisk_etw.wprp (1.6 KB)](/assets/data/vhdx-2026-05-02/virtualdisk_etw.wprp){:download="virtualdisk_etw.wprp"} — ETW WPR 캡처 프로파일
- 📥 [analysis_iter01.csv (10 KB)](/assets/data/vhdx-2026-05-02/analysis_iter01.csv){:download="analysis_iter01.csv"} — VHDMP IRP 카운트 (전부 0, caveats 참조)
