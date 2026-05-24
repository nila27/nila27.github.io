---
title: "NtCreateFile은 빌드마다 다르게 동작한다 — 두 케이스"
date: 2026-05-21 11:00:00 +0900
categories: [Windows Internals, File System]
tags: [ntcreatefile, kernel, disassembly, build-differences, windbg]
lang: ko-KR
---

> **3줄 요약**
> - `NtCreateFile`의 내부 호출 경로는 Windows 빌드마다 비자명하게 바뀐다.
> - 두 사례를 디스어셈블로 비교한다 — `[Case 1 짧은 제목]` / `[Case 2 짧은 제목]`.
> - 한 빌드만 본 분석은 *그 빌드의 사진*이다 — 인용할 땐 빌드 번호를 명시한다.
{: .prompt-tip }

`NtCreateFile`은 Windows 파일 시스템의 첫 syscall이다. 그 내부에 대한
분석은 인터넷에 많지만, 대부분은 *한 빌드를 본 시점의 스냅샷*이다.
실제 분석에서 가장 자주 부딪히는 어려움은 — *내가 보고 있는 빌드의
코드가, 인용된 글이 본 빌드의 코드와 다르다*는 사실이다.

이 글은 그 차이가 비자명하게 드러난 두 케이스를 본다.

분석 환경:

- 빌드 A: `[예: Windows 10 22H2 / 19045.xxxx]`
- 빌드 B: `[예: Windows 11 24H2 / 26100.xxxx]`
- 도구: WinDbg + Microsoft 공개 심볼, `uf /c`, `ntoskrnl.exe` 디스어셈블

---

## Case 1 — `[짧고 정보 밀도 높은 제목]`

`[케이스의 한 문장 요약 — 무엇이 어떻게 다른지]`

### 빌드 A — `[동작 요약 한 줄]`

```text
[디스어셈블 발췌, 10~20줄.
 analyze/ 디렉토리의 raw 파일에서 핵심 부분만 추려서 박는다.]
```
{: file='analyze/case1-buildA.txt' }

`[2~3문장으로 이 빌드의 동작을 풀이. "여기서 X를 부르고 Y로 분기" 식.]`

### 빌드 B — `[동작 요약 한 줄]`

```text
[디스어셈블 발췌, 10~20줄]
```
{: file='analyze/case1-buildB.txt' }

`[2~3문장으로 이 빌드의 동작을 풀이. 빌드 A와의 차이를 명시.]`

### 무엇이 바뀌었나

`[2~4문장. 함수 분리/합병, 인자 캡처 시점 이동, 분기 변경 등.
 가능하면 *왜* 바뀌었는지 추정도 한 줄 — 보안 mitigation? 성능?
 코드 정리?]`

### 분석자에게 의미

`[2~4문장. 이 차이가 트레이스 해석/디버깅/문서화에 어떻게 영향을 주는지.
 예: "한 빌드에서 잡힌 콜스택이 다른 빌드에선 한 프레임 사라진다",
 "Forshaw 글의 분석은 빌드 A 기준이라 빌드 B엔 다르게 적용해야 한다" 등.]`

---

## Case 2 — `[짧고 정보 밀도 높은 제목]`

`[케이스의 한 문장 요약]`

### 빌드 A — `[동작 요약 한 줄]`

```text
[디스어셈블 발췌]
```
{: file='analyze/case2-buildA.txt' }

`[풀이]`

### 빌드 B — `[동작 요약 한 줄]`

```text
[디스어셈블 발췌]
```
{: file='analyze/case2-buildB.txt' }

`[풀이]`

### 무엇이 바뀌었나

`[풀이]`

### 분석자에게 의미

`[풀이]`

---

## 한 가지만 기억한다면

같은 함수의 같은 이름이 *같은 동작*을 의미하지는 않는다. `NtCreateFile`
처럼 OS 핵심에 있는 함수일수록 빌드마다 내부가 바뀐다. 분석 글을 쓸 때
빌드 번호를 본문에 박는 이유다 — 글의 수명을 *내가 본 빌드의 수명*으로
한정한다는 의미고, 그 안에서만 정확하다는 약속이다.

---

## 첨부 자료

- 📥 [analyze/case1-buildA.txt](/analyze/case1-buildA.txt){:download="case1-buildA.txt"} — Case 1, 빌드 A raw 디스어셈블
- 📥 [analyze/case1-buildB.txt](/analyze/case1-buildB.txt){:download="case1-buildB.txt"} — Case 1, 빌드 B raw 디스어셈블
- 📥 [analyze/case2-buildA.txt](/analyze/case2-buildA.txt){:download="case2-buildA.txt"} — Case 2, 빌드 A
- 📥 [analyze/case2-buildB.txt](/analyze/case2-buildB.txt){:download="case2-buildB.txt"} — Case 2, 빌드 B

관련 글:

- [NtCreateFile, 세 번 분석하기][post-retrospective] — 이 분석에 도달한 과정의 메타 기록
- [Hyper-V vhdmp.sys 베이스라인][post-vhdx] — 같은 측정-우선 접근의 다른 사례

[post-retrospective]: /posts/ntcreatefile-three-analyses-ko/
[post-vhdx]: /posts/virtual-disk-benchmark-data-ko/
