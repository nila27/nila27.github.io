---
title: "Tracing vhdmp.sys: why your WPR session captures zero VHDMP events"
date: 2026-05-02 19:00:00 +0900
categories: [Cloud Storage, Hyper-V Disk]
tags: [etw, wpr, vhdmp, kernel, debugging]
lang: en
---

🇰🇷 [한국어 버전](/posts/vhdmp-etw-zero-events-ko/) ・ 🇺🇸 English (you are here)

> **TL;DR**
> - WPR-captured trace of the VHDMP provider returns zero per-IO events.
> - Cause: VHDMP routes its per-IO opcodes (200/201) only to the Analytic channel, not real-time ETW sessions.
> - Fix: `wevtutil sl Microsoft-Windows-VHDMP/Analytic /e:true` before WPR capture, disable after.
{: .prompt-tip }

Work on cloud storage at the kernel layer behind a Hyper-V class
hypervisor long enough and you'll want to count IRPs flowing through
`vhdmp.sys`: how many `VHD_START_IO` and `VHD_COMPLETE_IO` events does
the driver emit per user IO? What's the IRP amplification factor for a
differencing chain? How many BAT updates per first-write block?

The standard tool on Windows is ETW. Enable the
`Microsoft-Windows-VHDMP` provider in a WPR profile, capture a trace,
walk it with the standard `OpenTrace` / `ProcessTrace` callback API,
filter on the provider GUID, count opcodes 200 and 201.

The trace came back with **39,570,286 total events** — clearly recording
something. **0 of them were from VHDMP**.

```
ETL summary:
  total events in trace : 39570286
  VHDMP events total    : 0
  VHDMP outside phases  : start=0 complete=0 other=0
```
{: file='analysis_iter01.txt' }

Every per-phase row was zero:

```csv
format,phase,pattern,iotype,block_size,qd,user_ops,vhdmp_start_io,vhdmp_complete_io,vhdmp_other,amplification,duration_sec
VHD-Fixed,matrix,sequential,write,4096,1,65536,0,0,0,0,4.02663
VHD-Fixed,matrix,sequential,read,4096,1,65536,0,0,0,0,3.51775
VHD-Fixed,matrix,random,write,4096,1,65536,0,0,0,0,4.70157
... (every row has 0 VHDMP events)
```
{: file='analysis_iter01.csv' }

The provider GUID was correct. The WPR profile listed it. `wpr.exe`
exited cleanly. The trace file was 2.5 GB.

## What I had

Provider GUID, confirmed via `logman query providers`:

```cpp
const GUID kVhdmpProviderGuid =
    { 0xE2816346, 0x87F4, 0x4F85,
      { 0x95, 0xC3, 0x0C, 0x79, 0x40, 0x9A, 0xA8, 0x9D } };

constexpr UCHAR kOpcodeStartIo    = 200;  // VHD_START_IO
constexpr UCHAR kOpcodeCompleteIo = 201;  // VHD_COMPLETE_IO
```
{: file='etl_analyzer.cpp' }

The WPR profile, `virtualdisk_etw.wprp`, requested verbose-level events:

```xml
<EventProvider Id="EP_VHDMP" Name="e2816346-87f4-4f85-95c3-0c79409aa89d" Level="5">
  <Keywords>
    <Keyword Value="0xFFFFFFFFFFFFFFFF"/>
  </Keywords>
</EventProvider>
```
{: file='virtualdisk_etw.wprp' }

Level 5 verbose, keyword mask all-ones — should be everything the
provider can emit.

The trace consumer, walking the ETL with the standard ETW callback API:

```cpp
void WINAPI EventRecordCallback(PEVENT_RECORD ev) {
    AnalyzerCtx* ctx = (AnalyzerCtx*)ev->UserContext;
    ctx->totalEvents++;

    if (memcmp(&ev->EventHeader.ProviderId,
               &kVhdmpProviderGuid, sizeof(GUID)) != 0) {
        return;          // ignore other providers
    }
    ctx->totalVhdmpEvents++;
    // ... count opcode 200/201, attribute to phase ...
}
```
{: file='etl_analyzer.cpp' }

`totalEvents` reached 39.5M. `totalVhdmpEvents` stayed at 0. The walk
was correct — the trace itself didn't contain VHDMP events.

## Why

`Microsoft-Windows-VHDMP` is one of those Windows providers that route
their per-IO events **only to their Analytic event log channel**, not
to a real-time ETW session created via `wpr -start` or
`StartTraceW(EVENT_TRACE_REAL_TIME_MODE)`. The provider sees standard
ETW listeners and dutifully emits the non-Analytic events
(initialization, attach/detach, errors) — those events show up if you
trace it on a non-IO workload. But the two opcodes you actually want,
`VHD_START_IO` (200) and `VHD_COMPLETE_IO` (201), only emit when the
Analytic channel is enabled.

The Analytic channel is **disabled by default**. `wpr -start` doesn't
enable it. Your trace records every initialization event, every loader
event, every system-layer disk IO — but the actual VHDMP per-IO event is
silently dropped at the provider before it ever reaches your trace
session.

## The fix

Enable the channel before starting WPR:

```powershell
wevtutil.exe sl "Microsoft-Windows-VHDMP/Analytic" /e:true /q:true
```

Then `wpr -start virtualdisk_etw.wprp -filemode`. After
`wpr -stop trace.etl`, disable it again to stop disk write
amplification:

```powershell
wevtutil.exe sl "Microsoft-Windows-VHDMP/Analytic" /e:false /q:true
```

Automated in the benchmark wrapper:

```cpp
bool EnableEventLogChannel(const wchar_t* channelName, bool enable, ...) {
    wchar_t cmd[256];
    StringCchPrintfW(cmd, _countof(cmd),
        L"wevtutil.exe sl \"%s\" /e:%s /q:true",
        channelName, enable ? L"true" : L"false");
    return RunCommand(cmd, ...) == 0;
}

// usage:
// EnableEventLogChannel(L"Microsoft-Windows-VHDMP/Analytic", true, ...);
//   ← must be called before wpr -start, else trace will have zero VHDMP IRP events
```
{: file='etw_orchestrator.cpp' }

Two lines. Without them your trace burns gigabytes of disk and contains
none of the data you went there for.

> **Why two lines, not one?**
> The Analytic channel writes its own `.evtx` file by design — even with
> no real-time consumer, enabling it costs disk. Disable after capture
> unless you intend to keep the channel always-on.
{: .prompt-info }

## What still didn't work

Even with the Analytic channel enabled, the events end up in the
**channel's own `.evtx` log**, not the WPR `.etl` real-time stream.
You consume the `.evtx` separately — `Get-WinEvent`, the `EvtQuery` API,
or merging both streams in a custom analyzer. WPR records the provider,
but the routing for these specific opcodes still goes through the
channel.

For the benchmark report, the analyzer that walks the `.etl` keeps
emitting zeros, and per-IRP amplification has to be inferred indirectly
from `IOPS × duration` ratios. Not as clean as a direct count, but
workable.

## How I think about this at the cloud kernel layer

The cloud storage kernel work I do — virtual disk drivers, snapshot
stacks, storage proxies in front of `vhdmp.sys` — leans heavily on ETW
for IRP-level visibility. After hitting this trap, my worry: if CI
captures traces and an amplification chart silently goes to zero
overnight, what got shipped is a regression-checking pipeline that
doesn't actually check anything. From what I've seen, the
Analytic-channel routing trap looks like one of three or four similar
gotchas across Microsoft's storage providers, and the symptom (zero
events, otherwise valid trace) seems to repeat across the family.

## Pattern summary

For ETW traces from a Microsoft-defined provider whose per-event work is
"Analytic"-grade in volume:

1. Check whether the provider routes its key events to a channel:
   `logman query <provider>` and look for "Channels:".
2. If yes, `wevtutil sl <channel> /e:true` **before** the trace starts.
3. After capture, the events may still be in the `.evtx`, not the
   `.etl`. Read the right file.
4. Re-disable the channel after to avoid permanent disk write
   amplification.

Roughly half of Microsoft's diagnostic providers behave this way. The
docs rarely call it out — you find out by checking the trace and seeing
zeros.

---

## Attached data

- 📥 [virtualdisk_etw.wprp (1.6 KB)](/assets/data/vhdx-2026-05-02/virtualdisk_etw.wprp){:download="virtualdisk_etw.wprp"} — full WPR profile used in this post
- 📥 [analysis_iter01.csv (10 KB)](/assets/data/vhdx-2026-05-02/analysis_iter01.csv){:download="analysis_iter01.csv"} — VHDMP IRP count results (every row zero)

Related:
- [VHDX-Dynamic first-write penalty][post1] — original purpose of the ETW trace
- [Hyper-V vhdmp.sys baseline dataset][post2] — amplification estimated via the workaround

[post1]: /posts/vhdx-first-write-penalty-en/
[post2]: /posts/virtual-disk-benchmark-data-en/
