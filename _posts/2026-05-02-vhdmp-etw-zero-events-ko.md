---
title: "vhdmp.sys 추적: WPR 세션이 VHDMP 이벤트를 0개 잡는 이유"
date: 2026-05-02 19:00:00 +0900
categories: [Windows Analyze, FileSystem]
tags: [etw, wpr, vhdmp, kernel, debugging]
lang: ko-KR
---

🇰🇷 한국어 (현재) ・ 🇺🇸 [English version](/posts/vhdmp-etw-zero-events-en/)

> **3줄 요약**
> - VHDMP provider를 WPR로 캡처해도 per-IO 이벤트가 0개로 나오는 현상.
> - 원인: VHDMP는 per-IO opcode(200/201)를 Analytic 채널로만 라우팅, real-time ETW 세션엔 안 보냄.
> - 해결: `wevtutil sl Microsoft-Windows-VHDMP/Analytic /e:true` → wpr 캡처 → 다시 비활성화.
{: .prompt-tip }

Hyper-V 계열 hypervisor 뒤편의 클라우드 스토리지 커널 작업을 하다 보면
`vhdmp.sys`로 흐르는 IRP를 카운트하고 싶을 때가 있을 것이다. 사용자 IO
1개당 드라이버는 `VHD_START_IO`와 `VHD_COMPLETE_IO`를 몇 개 발행할까?
differencing chain의 IRP amplification factor는 얼마일까? 처음 쓸 때
블록당 BAT 업데이트는 몇 번일까?

Windows에서 이걸 측정하는 표준 도구는 ETW다. WPR 프로파일에
`Microsoft-Windows-VHDMP` provider를 활성화하고, 트레이스를 캡처해서
표준 `OpenTrace`, `ProcessTrace` 콜백 API로 walk하면서 provider GUID로
필터, opcode 200, 201을 카운트한다.

트레이스는 **39,570,286개의 이벤트**를 추적했다 — 뭔가는 분명히 기록된
것 같은데, 그중 **VHDMP 이벤트는 0개**였다.

```
ETL summary:
  total events in trace : 39570286
  VHDMP events total    : 0
  VHDMP outside phases  : start=0 complete=0 other=0
```
{: file='analysis_iter01.txt' }

분석의 모든 페이즈별 행이 0:

```csv
format,phase,pattern,iotype,block_size,qd,user_ops,vhdmp_start_io,vhdmp_complete_io,vhdmp_other,amplification,duration_sec
VHD-Fixed,matrix,sequential,write,4096,1,65536,0,0,0,0,4.02663
VHD-Fixed,matrix,sequential,read,4096,1,65536,0,0,0,0,3.51775
VHD-Fixed,matrix,random,write,4096,1,65536,0,0,0,0,4.70157
... (모든 행이 VHDMP 이벤트 0)
```
{: file='analysis_iter01.csv' }

provider GUID는 맞았다. WPR 프로파일에 들어 있었다. `wpr.exe`는 정상
종료했다. 트레이스 파일은 2.5 GB.

## 가지고 있던 것

provider GUID — `logman query providers`로 확인:

```cpp
const GUID kVhdmpProviderGuid =
    { 0xE2816346, 0x87F4, 0x4F85,
      { 0x95, 0xC3, 0x0C, 0x79, 0x40, 0x9A, 0xA8, 0x9D } };

constexpr UCHAR kOpcodeStartIo    = 200;  // VHD_START_IO
constexpr UCHAR kOpcodeCompleteIo = 201;  // VHD_COMPLETE_IO
```
{: file='etl_analyzer.cpp' }

WPR 프로파일 `virtualdisk_etw.wprp`는 이 provider의 verbose 레벨 이벤트를
요청:

```xml
<EventProvider Id="EP_VHDMP" Name="e2816346-87f4-4f85-95c3-0c79409aa89d" Level="5">
  <Keywords>
    <Keyword Value="0xFFFFFFFFFFFFFFFF"/>
  </Keywords>
</EventProvider>
```
{: file='virtualdisk_etw.wprp' }

Level 5는 verbose. Keyword mask는 all-ones. Provider가 발행할 수 있는
모든 걸 받아야했다.

표준 ETW 콜백 API로 ETL을 walk하는 컨슈머:

```cpp
void WINAPI EventRecordCallback(PEVENT_RECORD ev) {
    AnalyzerCtx* ctx = (AnalyzerCtx*)ev->UserContext;
    ctx->totalEvents++;

    if (memcmp(&ev->EventHeader.ProviderId,
               &kVhdmpProviderGuid, sizeof(GUID)) != 0) {
        return;          // 다른 provider 이벤트는 무시
    }
    ctx->totalVhdmpEvents++;
    // ... opcode 200/201 카운트, phase에 attribute ...
}
```
{: file='etl_analyzer.cpp' }

`totalEvents`는 39.5M에 도달. `totalVhdmpEvents`는 0에 머무름. walk는
정확했다 — 트레이스 자체에 VHDMP 이벤트가 들어있지 않았던 것이다.

## 이유

`Microsoft-Windows-VHDMP`는 **per-IO 이벤트를 자기 Analytic 이벤트 로그
채널로만 라우팅**하고, `wpr -start`나 `StartTraceW(EVENT_TRACE_REAL_TIME_MODE)`로
만든 real-time ETW 세션엔 보내지 않는, 그런 부류의 Windows provider 중
하나다. Provider는 표준 ETW 경로의 listener들을 인지해 *비*-Analytic
이벤트(초기화, attach/detach, 오류)는 충실히 발행한다 — 이 이벤트들은
*비-IO 워크로드*에 트레이스를 걸면 보인다. 정작 원하는 두 opcode,
`VHD_START_IO`(200)과 `VHD_COMPLETE_IO`(201)은 Analytic 채널이 활성화된
경우에만 발행된다.

Analytic 채널은 **기본 비활성**. `wpr -start`가 알아서 켜주지 않는다.
트레이스는 모든 초기화 이벤트, 모든 loader 이벤트, 모든 system-layer
디스크 IO를 기록하지만 — 정작 원했던 VHDMP per-IO 이벤트는 트레이스
세션에 닿기도 전에 provider에서 조용히 떨어진다.

## 해결

WPR을 시작하기 *전에* 채널을 활성화:

```powershell
wevtutil.exe sl "Microsoft-Windows-VHDMP/Analytic" /e:true /q:true
```

그 후 `wpr -start virtualdisk_etw.wprp -filemode`. `wpr -stop trace.etl`
후엔 디스크 write amplification을 막기 위해 채널을 다시 비활성화:

```powershell
wevtutil.exe sl "Microsoft-Windows-VHDMP/Analytic" /e:false /q:true
```

벤치마크 래퍼에서는 자동화한다:

```cpp
bool EnableEventLogChannel(const wchar_t* channelName, bool enable, ...) {
    wchar_t cmd[256];
    StringCchPrintfW(cmd, _countof(cmd),
        L"wevtutil.exe sl \"%s\" /e:%s /q:true",
        channelName, enable ? L"true" : L"false");
    return RunCommand(cmd, ...) == 0;
}

// 사용 시점:
// EnableEventLogChannel(L"Microsoft-Windows-VHDMP/Analytic", true, ...);
//   ← 이걸 wpr -start 전에 호출해야 VHDMP IRP 이벤트가 트레이스에 들어감
```
{: file='etw_orchestrator.cpp' }

두 줄. 이 두 줄이 없으면 트레이스는 디스크에 기가바이트 단위로 쌓이는데
정작 보러 갔던 데이터는 0개다.

> **왜 두 줄이고 한 줄이 아닐까?**
> Analytic 채널은 설계상 자체 `.evtx` 파일에 쓴다 — real-time consumer가
> 없어도 채널이 켜져 있으면 디스크 비용이 든다. 채널을 항상-켜둘 의도가
> 아니면 캡처 후 항상 비활성화한다.
{: .prompt-info }

## 여전히 안 된 것

Analytic 채널을 켜도, 이벤트는 결국 **채널 자체의 `.evtx` 로그**에
들어가지 WPR `.etl` real-time 스트림에는 들어가지 않는다. `Get-WinEvent`,
`EvtQuery` API, 또는 두 스트림을 머지하는 커스텀 분석기로 `.evtx`를 별도
소비해야 한다. WPR이 provider를 기록하긴 하지만, 이 특정 opcode들의
라우팅은 여전히 채널을 통해 간다.

벤치마크 리포트에서는 결국 `.etl`을 walk하는 분석기가 계속 0을 내보냈고,
per-IRP amplification은 `IOPS × duration` 비율에서 간접 추정해야 했다.
직접 카운트만큼 깔끔하진 않지만 쓸 만하다.

## 클라우드 커널 레이어에서 이걸 어떻게 볼까

내가 해 온 클라우드 스토리지 커널 작업 — 가상 디스크 드라이버, 스냅샷
스택, `vhdmp.sys` 앞에 앉는 스토리지 프록시 — 에서는 IRP-level 가시성을
위해 ETW에 크게 기댄다. 이번에 트랩에 걸리고 든 걱정은 이거다: CI가
트레이스를 캡처하는데 amplification 차트가 어느 날 밤 조용히 0으로
떨어지면, 결국 아무것도 검증하지 않는 회귀 검사 파이프라인을 출고한
셈이다. 내가 부딪힌 범위에서는 Analytic-채널 라우팅 트랩이 Microsoft
스토리지 provider들에 있는 비슷한 함정 3~4개 중 하나로 보이고, 증상(이벤트
0, 나머지는 정상 트레이스)도 비슷한 부류 전반에서 반복되는 듯하다.

## 패턴 요약

per-event 분량이 "Analytic" 등급인 Microsoft 정의 provider의 ETW 트레이스를
캡처할 때:

1. provider의 핵심 이벤트가 채널로 라우팅되는지 확인:
   `logman query <provider>`에서 "Channels:" 항목.
2. 그렇다면 트레이스 시작 **전에** `wevtutil sl <channel> /e:true`.
3. 캡처 후 이벤트는 여전히 `.evtx`에 있을 수 있음. 맞는 파일을 읽는다.
4. 캡처 후 채널을 다시 비활성화 — 영구 디스크 write amplification 방지.

Microsoft의 진단 provider 중 절반 정도가 이렇게 동작한다. 문서가 명시적으로
짚어주는 일은 드물다 — 트레이스를 보고 0을 발견함으로써 알아낸다.

---

## 첨부 자료

- 📥 [virtualdisk_etw.wprp (1.6 KB)](/assets/data/vhdx-2026-05-02/virtualdisk_etw.wprp){:download="virtualdisk_etw.wprp"} — 본문에서 사용한 WPR 프로파일 전체
- 📥 [analysis_iter01.csv (10 KB)](/assets/data/vhdx-2026-05-02/analysis_iter01.csv){:download="analysis_iter01.csv"} — VHDMP IRP 카운트 결과 (모든 행이 0)

관련 글:
- [VHDX-Dynamic 첫쓰기 페널티][post1] — 본 ETW 캡처의 원래 목적
- [Hyper-V vhdmp.sys 베이스라인 데이터][post2] — 우회 추정으로 얻은 amplification

[post1]: /posts/vhdx-first-write-penalty-ko/
[post2]: /posts/virtual-disk-benchmark-data-ko/
