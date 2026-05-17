---
title: "Windows Update 분석을 위한 데이터 수집 베이스라인 (PowerShell)"
date: 2026-05-17 10:00:00 +0900
categories: [WindowsUpdate, Collection]
tags: [powershell, winsxs, audit, baseline, kb]
lang: ko-KR
---

> **3줄 요약**
> - 향후 KB·빌드별 Windows Update 분석 글에 쓸 데이터의 수집 베이스라인.
> - PowerShell 한 파일로 KB 목록 + WinSxS 변경 바이너리(Old/Latest) + SHA256 + WindowsUpdate 이벤트를 한 폴더에 떨군다.
> - **수집은 부분집합이다** — System32, CBS 로그, Analytic 채널, 컴포넌트 manifest는 들어있지 않다. 수집 방법은 추후 개선 예정.
{: .prompt-tip }

Windows Update를 KB/빌드 단위로 분석하는 글을 시리즈로 쓰려고 한다.
분석을 시작하기 전에 *어떤 빌드에서 무엇이 바뀌었는지* 를 매번 다시
찾는 건 비용이 크다. 한 번 수집해 두면 그 다음부터의 분석 글은 항상
이 스냅샷 위에서 출발할 수 있다.

이 글은 그 스냅샷을 만드는 PowerShell 스크립트와, 무엇을 일부러
빼두었는지에 대한 기록이다.

## 수집 대상

한 번 실행으로 떨구는 것:

| 항목 | 출처 | 형식 |
| --- | --- | --- |
| 설치된 KB 목록 | `Win32_QuickFixEngineering` | CSV |
| 최근 N일간 바뀐 바이너리 (Old/Latest) | `C:\Windows\WinSxS` 스캔 | 파일 복사본 |
| 위 파일들의 SHA256 | `Get-FileHash` | CSV |
| 수집된 파일 메타데이터 | 위 스캔 결과 | CSV |
| WindowsUpdate provider 이벤트 | `System` 이벤트 로그 | CLIXML |

기본값은 `$days = 30`. 한 달 주기로 돌리면 직전 패치 사이클에서 만진
바이너리가 잡힌다.

## 스크립트

`Collect-WindowsUpdateAudit.ps1` 한 파일. 관리자 권한으로 실행한다.

```powershell
$days = 30

$base = "$env:USERPROFILE\Downloads\WindowsUpdateAudit"
$latestDir = Join-Path $base "latest"
$oldDir = Join-Path $base "old"
$reportDir = Join-Path $base "reports"
$logDir = Join-Path $base "logs"

New-Item -ItemType Directory -Force -Path $base | Out-Null
New-Item -ItemType Directory -Force -Path $latestDir | Out-Null
New-Item -ItemType Directory -Force -Path $oldDir | Out-Null
New-Item -ItemType Directory -Force -Path $reportDir | Out-Null
New-Item -ItemType Directory -Force -Path $logDir | Out-Null

Write-Host ""
Write-Host "Collecting installed KB updates..."

Get-CimInstance Win32_QuickFixEngineering |
Sort-Object InstalledOn -Descending |
Export-Csv (
    Join-Path $reportDir "installed_kbs.csv"
) -NoTypeInformation -Encoding UTF8

Write-Host "Scanning updated binaries..."

$files = Get-ChildItem "C:\Windows\WinSxS" -Recurse -File `
    -Include *.dll,*.sys,*.exe `
    -ErrorAction SilentlyContinue |
Where-Object {
    $_.LastWriteTime -gt (Get-Date).AddDays(-$days)
}

$grouped = $files | Group-Object Name

$result = @()
$hashes = @()

foreach ($group in $grouped) {

    $sorted = $group.Group | Sort-Object {
        try {
            [version]$_.VersionInfo.FileVersion
        }
        catch {
            [version]"0.0.0.0"
        }
    }

    if ($sorted.Count -eq 1) {

        $latest = $sorted[-1]

        $dest = Join-Path $latestDir $latest.Name

        try {

            Copy-Item $latest.FullName $dest -Force

            $hash = Get-FileHash $dest -Algorithm SHA256

            $result += [pscustomobject]@{
                File = $latest.Name
                Type = "LatestOnly"
                Version = $latest.VersionInfo.FileVersion
                Source = $latest.FullName
                CopiedTo = $dest
            }

            $hashes += [pscustomobject]@{
                File = $latest.Name
                Type = "LatestOnly"
                SHA256 = $hash.Hash
            }

        } catch {}
    }
    else {

        $old = $sorted[0]
        $latest = $sorted[-1]

        $oldDest = Join-Path $oldDir $old.Name
        $latestDest = Join-Path $latestDir $latest.Name

        try {

            Copy-Item $old.FullName $oldDest -Force
            $oldHash = Get-FileHash $oldDest -Algorithm SHA256

            $result += [pscustomobject]@{
                File = $old.Name
                Type = "Old"
                Version = $old.VersionInfo.FileVersion
                Source = $old.FullName
                CopiedTo = $oldDest
            }

            $hashes += [pscustomobject]@{
                File = $old.Name
                Type = "Old"
                SHA256 = $oldHash.Hash
            }

        } catch {}

        try {

            Copy-Item $latest.FullName $latestDest -Force
            $latestHash = Get-FileHash $latestDest -Algorithm SHA256

            $result += [pscustomobject]@{
                File = $latest.Name
                Type = "Latest"
                Version = $latest.VersionInfo.FileVersion
                Source = $latest.FullName
                CopiedTo = $latestDest
            }

            $hashes += [pscustomobject]@{
                File = $latest.Name
                Type = "Latest"
                SHA256 = $latestHash.Hash
            }

        } catch {}
    }
}

Write-Host "Exporting reports..."

$result |
Sort-Object File,Type |
Export-Csv (
    Join-Path $reportDir "collected_files.csv"
) -NoTypeInformation -Encoding UTF8

$hashes |
Sort-Object File,Type |
Export-Csv (
    Join-Path $reportDir "hashes.csv"
) -NoTypeInformation -Encoding UTF8

Get-WinEvent -LogName System -MaxEvents 300 |
Where-Object {
    $_.ProviderName -match "WindowsUpdate"
} |
Export-Clixml (
    Join-Path $logDir "windows_update_events.xml"
)

Write-Host ""
Write-Host "Completed."
Write-Host ""
Write-Host "Output:"
Write-Host $base
Write-Host ""
```
{: file='Collect-WindowsUpdateAudit.ps1' }

## 출력 구조

```
Downloads
 └─ WindowsUpdateAudit
     ├─ latest                          # 그룹별 최신 버전 바이너리 사본
     ├─ old                             # 그룹별 이전 버전 바이너리 사본
     ├─ reports
     │   ├─ installed_kbs.csv           # 설치된 KB 목록
     │   ├─ collected_files.csv         # 수집된 파일의 메타데이터
     │   └─ hashes.csv                  # SHA256
     └─ logs
         └─ windows_update_events.xml   # System 로그의 WU provider 이벤트
```

이 폴더 하나가 한 시점의 *분석 가능한 스냅샷*이다. 다음에 KB가 설치된
직후 다시 돌리면 새 폴더를 따로 두고 두 스냅샷을 비교할 수 있다.

## 동작 원리

WinSxS는 컴포넌트 스토어다. KB가 설치되면 새 버전의 페이로드가 이
디렉터리 아래에 추가되고, System32에 노출되는 바이너리는 그 페이로드로
재링크된다. *최근 N일간 LastWriteTime이 갱신된 바이너리*를 모으면
"이번 패치 사이클에 바뀐 것들"의 후보 집합이 만들어진다.

같은 이름의 DLL이 여러 버전 잡히면 `FileVersion` 기준으로 정렬해서
가장 낮은 걸 `old`, 가장 높은 걸 `latest`로 분리한다.

## 지금은 빠져 있는 것

수집 자체에 너무 많은 시간을 쓰지 않으려고 우선 최소 집합부터 끊었다.
아래 항목들은 의미가 없어 뺀 게 아니라, 한 번에 다 잡으려면 스크립트가
훨씬 무거워지기 때문에 어쩔 수 없이 이번 수집에선 누락됐다.

- **System32 / SysWOW64 스냅샷.** Windows가 실제로 로드하는 바이너리는
  WinSxS 페이로드로 하드링크된 System32 쪽이다. WinSxS만 봐도 *어떤
  버전이 설치되어 있는지*는 알지만 *지금 로드되는 게 어느 버전인지*는
  엄밀히 말할 수 없다.
- **CBS 로그.** `C:\Windows\Logs\CBS\CBS.log`는 컴포넌트 설치/철회의
  핵심 기록이다. 텍스트 포맷이라 파싱 코드가 별도로 붙어야 해서
  이번엔 같이 못 가져왔다.
- **Analytic 채널의 WU 이벤트.** 지금 잡는 건 `System` 로그의
  `WindowsUpdate` provider 이벤트뿐이다.
  `Microsoft-Windows-WindowsUpdateClient/Operational` 채널과
  Analytic 채널은 별도다 — 이쪽은 [vhdmp 글][vhdmp-etw]에서 다룬
  것과 같은 패턴으로, 채널을 먼저 켜고 캡처해야 한다.
- **컴포넌트 manifest (`*.manifest`).** WinSxS 아래 manifest는
  어느 DLL이 어느 컴포넌트에 속하는지를 알려주는 권위 있는 출처다.
  같이 떠올 수는 있지만 양이 많아 이번 수집에선 바이너리만 우선 챙겼다.
- **그룹화의 정밀도.** 지금은 `Group-Object Name`으로 묶는다. 같은
  파일명이 amd64·wow64·x86 또는 SKU별로 별도 페이로드에 존재할 수
  있는데, 이게 한 그룹에 섞이면 "Old vs Latest" 판별이 흐려진다.
  실제로 돌려보면 대부분 amd64 내 버전 차이로 들어맞지만,
  엄밀하지는 않다.

수집 방법은 추후 개선 예정이다.

## 수집 용량

내 환경 기준 한 번 수집하면 `WindowsUpdateAudit` 폴더 크기가 대게 **3~4 GB**
정도 된다. 대부분은 `latest`/`old`에 복사된 WinSxS 바이너리가 차지한다.
`$days` 값을 줄이거나 늘리면 비례해서 바뀐다. 분석 끝난 스냅샷은
별도 보관소로 옮기거나 압축해 두는 편이 좋다.

[vhdmp-etw]: /posts/vhdmp-etw-zero-events-ko/
