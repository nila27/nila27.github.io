---
title: "VHDX-Dynamic의 32 MiB 블록 기본값 — 클라우드 스토리지 커널의 함정"
date: 2026-05-02 18:00:00 +0900
categories: [Cloud Storage, Hyper-V Disk]
tags: [vhdmp, vhdx, kernel, hyper-v, block-storage]
lang: ko-KR
---

🇰🇷 한국어 (현재) ・ 🇺🇸 [English version](/posts/vhdx-first-write-penalty-en/)

> **3줄 요약**
> - 5종 가상 디스크 포맷 중 VHDX-Dynamic만 첫쓰기에서 15× 느림 (21 vs 318 IOPS).
> - 원인은 포맷이 아니라 `BlockSizeInBytes` 기본값(32 MiB) — VHDX-Differencing(2 MiB, 같은 엔진)이 VHD-Dynamic과 동등 성능.
> - 호스트/커널 측에서 새 디스크에 prefill할 땐 `BlockSizeInBytes = 2 * 1024 * 1024` 명시 오버라이드.
{: .prompt-tip }

Hyper-V 계열 hypervisor 뒤에서 도는 커널 코드 — 가상 디스크 드라이버,
클라우드 스토리지 할당기, 스냅샷 엔진 — 를 만지다 보면 결국 `vhdmp.sys`와
VHDX 포맷을 만난다. 사내 문서들은 보통 VHD-Fixed, VHD-Dynamic, VHDX-Fixed,
VHDX-Dynamic, VHDX-Differencing 다섯 포맷의 선택을 *의미 있는 성능 결정*
으로 다룬다. 끝까지 측정해 보면 통념은 거의 사라진다 — warm-up 이후엔
다섯 포맷이 모든 워크로드에서 5% 이내로 수렴한다.

예외는 첫 쓰기다. 각 블록을 처음 만지는 4 KiB 순차 쓰기 워크로드에서
VHDX-Dynamic은 **21 IOPS**, VHD-Dynamic은 **318 IOPS**. 업그레이드라던
VHDX가 15배 느렸다.

| 포맷                | 첫쓰기 IOPS  | p50    | p99      | max       |
| ------------------ | -----------: | -----: | -------: | --------: |
| VHD-Fixed          | (사전 할당)   | —      | —        | —         |
| VHD-Dynamic        |        318.2 |  124us |   67.9ms |   150.8ms |
| VHDX-Fixed         | (사전 할당)   | —      | —        | —         |
| **VHDX-Dynamic**   |     **21.1** |  283us |  151.7ms |   210.7ms |
| VHDX-Differencing  |        317.1 |  1.2ms |   12.5ms |    54.2ms |

포맷이 망가진 게 아니다. 두 포맷의 *유일하게 의미 있는 차이*는, 거의
누구도 건드리지 않는 한 가지 기본값에서 온다.

## 아무도 설정하지 않는 기본값

VHDX-Dynamic을 `CreateVirtualDisk`로 만들 때 API가 자동으로 고르는
`BlockSizeInBytes`는 **32 MiB**다. VHD-Dynamic은 2 MiB. 벤치마크
래퍼에서 직접 보면:

```cpp
ULONGLONG DefaultBlockSize(DiskFormat f) {
    switch (f) {
    case DiskFormat::VhdDynamic:       return 2ULL  * 1024 * 1024;  //  2 MiB
    case DiskFormat::VhdxDynamic:      return 32ULL * 1024 * 1024;  // 32 MiB ← 16배 큼
    case DiskFormat::VhdxDifferencing: return 2ULL  * 1024 * 1024;  //  2 MiB
    default:                            return 0;                   // Fixed: 해당 없음
    }
}
```
{: file='vdisk_factory.cpp' }

첫쓰기 페이즈는 2 GiB IO 범위에서 모든 블록의 첫 바이트, 이어서 두 번째
바이트에 4 KiB 쓰기를 발행한다. VHD-Dynamic은 약 1024개의 2 MiB 블록을
할당하면 끝난다. VHDX-Dynamic은 64개의 32 MiB 블록을 할당해야 하고 — 그
4 KiB 쓰기가 디스크에 닿기 전에 *블록당 32 MiB를 0으로 채워야* 한다.
블록당 16배 커진 zero-fill에 VHDX의 다소 무거운 메타데이터 경로가
곱해져, 측정된 15배가 나온다.

이 동작을 일으키는 실제 호출:

```cpp
CREATE_VIRTUAL_DISK_PARAMETERS p = {};
p.Version = CREATE_VIRTUAL_DISK_VERSION_2;
p.Version2.MaximumSize           = sizeBytes;
p.Version2.BlockSizeInBytes      = 0;     // ← 0이 "기본값 사용"
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

`BlockSizeInBytes = 0`이 함정이다. MSDN 샘플을 그대로 복사해온 코드 대부분이
이 값을 0으로 둔다. OS가 포맷 기본값을 끼워 넣는데, VHD-Dynamic에선
무난하지만 VHDX-Dynamic에선 이 워크로드를 파괴적으로 만든다.

## 결정적 증거

세 번째 포맷 — VHDX-Differencing — 이 결정적 증거다. 같은 VHDX 엔진,
같은 커널 드라이버 경로(`vhdmp.sys`), 같은 메타데이터 레이아웃. 다만
기본 블록 크기가 **2 MiB**. 그리고 첫쓰기 결과(317 IOPS)는 VHD-Dynamic과
사실상 동일하다.

변수는 포맷이 아니라 블록 크기다.

## 그럼 기본값이 잘못된 걸까

아니다. VHDX-Dynamic의 32 MiB 기본값은 **게스트 VM 워크로드**용으로
선택됐다. 거기서는 할당이 OS 설치 또는 첫 부팅 시점에 일어나 VM 수명
전반에 분산되고, 더 작은 블록은 BAT 크기와 정상 운영 중 메타데이터
churn을 늘린다. 게스트 VM에는 옳은 선택이다.

호스트 측 스크립트에는 — 그리고 커널 스토리지 경로에서 새로 할당된 클라우드
디스크에 prefill·복사·벤치마크를 돌리는 누구에게도 — 잘못된 선택이다.
다만 *그 워크로드는 애초에 이 기본값의 타겟이 아니었다*.

오버라이드는 한 줄:

```cpp
p.Version2.BlockSizeInBytes = 2 * 1024 * 1024;  // 명시적으로 2 MiB
```

`BlockSizeInBytes = 2 MiB`로 VHDX-Dynamic의 첫쓰기 처리량은 VHD-Dynamic
대비 몇 % 안에 들어온다. VHDX의 본질적 가치(>2 TB 용량, 4 KiB 섹터 정렬,
로그 기반 손상 복원)는 그대로 가져가면서, 다른 워크로드용 기본값을 위한
비용은 사라진다.

## 클라우드 스토리지 커널 관점에서 왜 신경쓸까

- **Provisioning 함대**: 클라우드 규모에서 VHDX-backed 디스크를 수천 개
  찍어낸다. 프로비저닝 커널 경로가 기본 블록 크기를 쓰고 새 디스크에
  즉시 메타데이터(signature, MBR, fs)를 4 KiB로 쓰면, *디스크당 한 번씩*
  이 페널티를 낸다 — 함대 크기만큼 곱해진다.
- **Backup 복원**: 스냅샷에서 VHDX를 복원하면 첫 쓰기에서 블록이
  할당된다. 복원 경로가 4 KiB 단위로 데이터를 흘리면, 아무도 경고 안
  했던 페널티를 커널 레이어에서 만난다.
- **스냅샷 패브릭으로서의 VHDX-Differencing**: 클라우드 스냅샷의
  copy-on-write에 가장 가까운 프리미티브라고 본다. Differencing의 첫쓰기가
  21이 아닌 317 IOPS라는 사실이 — 부모가 이미 존재하는 클론·스냅샷
  시나리오에서 — VHDX-Dynamic 대신 Differencing을 *고를 이유*다.
- **Steady state는 무관**: 디스크가 완전히 쓰여진 뒤엔 다섯 포맷이
  per-IO 지표 어디서나 5% 이내로 모인다.

## Steady-state 검증

| 워크로드 (4K rand R QD32) | IOPS  |
| ------------------------- | ----: |
| VHD-Fixed                 | 84.8K |
| VHD-Dynamic               | 84.2K |
| VHDX-Fixed                | 84.3K |
| VHDX-Dynamic              | 82.2K |
| VHDX-Differencing         | 84.3K |

다섯 포맷 모두 4% 이내. warm-up 이후 포맷은 디스크 위 *형상*에 영향을
주지 *런타임 비용*에는 영향을 주지 않는다.

## 한 가지만 기억한다면

**다섯 가상 디스크 포맷을 구분하는 유일한 성능 수치는 첫쓰기 페널티이고,
이건 전적으로 `BlockSizeInBytes`의 함수다 — 포맷 자체의 함수가 아니다.**
이걸 받아들이고 나면 *"VHD를 쓸까 VHDX를 쓸까"*는 더 이상 성능 문제가
아니라 기능 문제가 된다.

---

## 첨부 자료

- 📥 [aggregate.csv (22 KB)](/assets/data/vhdx-2026-05-02/aggregate.csv){:download="aggregate.csv"} — 5회 반복 셀별 평균 / 표준편차
- 📥 [report.txt (22 KB)](/assets/data/vhdx-2026-05-02/report.txt){:download="report.txt"} — 사람이 읽기 좋은 종합 리포트
- 📥 [virtualdisk_etw.wprp (1.6 KB)](/assets/data/vhdx-2026-05-02/virtualdisk_etw.wprp){:download="virtualdisk_etw.wprp"} — ETW 캡처 프로파일

관련 글:
- [Hyper-V vhdmp.sys 베이스라인: 5 포맷 × 158 워크로드 (데이터)](/posts/virtual-disk-benchmark-data-ko/)
- [vhdmp.sys 추적: WPR 세션이 VHDMP 이벤트를 0개 잡는 이유](/posts/vhdmp-etw-zero-events-ko/)
