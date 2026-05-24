---
title: "NtCreateFile, 세 번 분석하기"
date: 2026-05-21 10:00:00 +0900
categories: [Reflections]
tags: [ntcreatefile, kernel, filter-manager, windbg, reactos, retrospective]
lang: ko-KR
---

🇰🇷 한국어 (현재) ・ 🇺🇸 [English version](/posts/ntcreatefile-three-analyses-en/)

> **3줄 요약**
> - 같은 `NtCreateFile`을 대학생·주니어·현재 세 시점에 다시 분석했다.
> - 매번 새로 보인 것은 함수가 아니라 *내가 의미를 어디에서 가져오는가* 였다 — 도구 → 다른 사람의 글 → 1차 자료.
> - 분석 능력의 성장은 도구가 좋아진 게 아니라, 외부 해석에 의존하지 않게 되는 과정이었다.
{: .prompt-tip }

`NtCreateFile`은 Windows 파일 시스템의 첫 syscall이고, 모든
파일·디바이스·named pipe·mailslot 오픈의 진입점이다. 이 함수 자체를
다루는 분석은 이미 인터넷에 많다. 이 글은 그런 분석을 한 편 더 보태려는
게 아니라, *내가 같은 함수를 세 시기에 어떻게 다르게 봤는지*를 기록한다.

같은 대상을 시간 위에서 다시 보면, 새로 보이는 것은 함수가 아니라
이전의 내가 *무엇을 못 봤는지* 다.

## 시기 1 — 대학생: 심볼이 알려주는 이름

도구는 WinDbg, 심볼은 Microsoft 공개 심볼 서버. `ntdll.dll`의
`NtCreateFile`에 브레이크 걸고, `kb`로 콜스택을 보고, step-in으로
타고 들어가면서 *어떤 함수들이 호출되는지* 를 종이에 옮겨 적었다.

그때 그린 그림은 대략 이런 형태였다:

```
NtCreateFile
└─ IoCreateFile (또는 IoCreateFileEx)
   └─ ObOpenObjectByName (...)
      └─ ObpLookupObjectName (...)
         └─ IopParseDevice (...)
            └─ IofCallDriver → IRP_MJ_CREATE → (드라이버 함수)
```
{: file='notes-univ.txt' }

이 그림은 *틀리지 않았다*. 호출 관계는 실제로 이렇다. 다만 이 그림이
내가 그 시기에 본 *전부*였다. 각 함수 옆에 무엇을 적었느냐가 핵심이다 —
나는 **함수 이름만 적었다**. `ObpLookupObjectName`이 *무엇을 하는지*
나는 몰랐고, 알아볼 1차 자료를 어디에서 찾아야 하는지도 몰랐다.

심볼이 알려주는 건 *그 함수가 존재한다*는 사실 뿐이다. *무엇을 하는지*
는 알려주지 않는다. 도구가 답해줄 수 있는 질문과 그렇지 않은 질문의
경계를, 나는 그때 인지하지 못했다.

이 시기의 한계는 단순했다 — **call graph는 봤지만, semantic은 비어
있었다.**

## 시기 2 — 주니어: 다른 사람의 글이 의미를 채워주다

두 가지가 동시에 들어왔다.

하나는 Project Zero에 올라온 [James Forshaw의 글][forshaw1]이었다.
`NtCreateFile`을 *보안 관점*에서 — `PreviousMode`, `AccessMode`,
캡처된 인자의 검증 시점 — 다루는 그 글은, 내가 대학생 시절 종이에
"`ObpLookupObjectName`" 이라고만 적어 두었던 박스 옆에 *그 함수가
신뢰하는 입력이 무엇인지* 를 처음으로 채워줬다.

다른 하나는 [ReactOS 소스][reactos]였다. NT 커널의 오픈소스 재구현은
공식 Windows 소스가 아니지만, *의도가 같은 코드*를 읽을 수 있게 해줬다.
Forshaw의 글이 "이런 의미"라고 말한 부분을, ReactOS의 `IoCreateFile` /
`ObpLookupObjectName` 구현에서 *대응하는 코드*로 확인할 수 있었다.

이 두 자료를 옆에 두고 다시 그린 그림은, 함수 이름 옆에 한 줄씩
주석이 붙은 형태였다:

```
NtCreateFile
└─ IoCreateFile
   ├─ AccessMode/PreviousMode 캡처            ← 누가 호출했는지가 여기서 결정됨
   ├─ OBJECT_ATTRIBUTES 캡처/검증              ← user 메모리에서 커널로 안전 복사
   └─ ObOpenObjectByName
      └─ ObpLookupObjectName
         ├─ namespace traversal               ← \??\, \Device, ...
         ├─ symbolic link / reparse           ← 같은 syscall로 재진입 가능
         └─ IopParseDevice
            ├─ FILE_OBJECT 할당
            ├─ Security check                 ← 어느 mode로?  ← Forshaw 글의 핵심
            └─ IRP_MJ_CREATE → driver
```
{: file='notes-junior.txt' }

각 함수 옆에 *무엇을 하는지* 가 한 줄씩 적혔다는 점이 대학생 시절과의
유일한 차이였다. call graph는 같았다. 채워진 건 semantic이었다.

이 시기의 진짜 능력은 **읽기**였다. 누군가 써둔 글과 누군가 구현해둔
코드를 읽고 *내 노트로 옮기는* 능력. 종속적이지만 강력하다 — 이 단계가
없으면 다음 단계는 불가능하다.

여전히 안 보이는 것은 있었다. Forshaw의 글이 다루지 않은 부분 —
ECP(Extra Create Parameters) 체인, Filter Manager가 어느 지점에서
끼어드는지, oplock과 transaction의 상호작용 — 은 노트에 빈 칸으로
남아 있었다. 빈 칸인 줄도 몰랐다. *누가 글을 써주지 않으면 그 박스가
존재한다는 것조차 모르는 상태*였다.

[forshaw1]: https://googleprojectzero.blogspot.com/2019/03/windows-kernel-logic-bug-class-access.html
[reactos]: https://github.com/reactos/reactos

## 시기 3 — 현재: 1차 자료에서 직접

이 시기에는 의지하는 자료의 종류가 바뀌었다. 더 정확히는, *해석된
자료에서 1차 자료로* 이동했다.

- **디스어셈블**: `ntoskrnl.exe`와 `fltmgr.sys`를 직접 읽는다. 심볼이
  알려주는 것은 함수의 존재이고, 디스어셈블이 알려주는 것은 함수의
  *행동*이다. 시기 1의 도구를 시기 2의 능력으로 다시 쓰는 셈이다.
- **공개 헤더와 공식 문서**: `wdm.h`, `ntifs.h`, `fltKernel.h`. 의도가
  주석으로 박혀 있다.
- **실측**: 디스어셈블이 *말하는* 동작을, 실제 트레이스로 *확인*한다.
  ETW, WinDbg conditional breakpoint, `!fileobj`, `!devstack`.

이 도구들의 가치는 *움직이는 표적을 따라갈 수 있다*는 점에서 가장
선명하게 드러난다. Project Zero의 글은 그 글이 쓰여진 시점의 빌드를
본 것이고, ReactOS는 특정 NT 버전을 향한 재구현이다. 둘 다 시간 위에서
굳은 사진이다 — *내가 지금 보고 있는 빌드*가 그 사진과 같다는 보장은
없다.

시기 3이 실제로 작동하는 한 사례는 별도 글로 다뤘다 —
[NtCreateFile은 빌드마다 다르게 동작한다 — 두 케이스][post-builddiff].
그 글이 시기 3의 작동 방식을 *코드로* 보여주는 자료다.

이 시기의 핵심은 *능력*보다 *인지*에 있다. 노트의 어디가 빈 칸인지를
스스로 알아본다. 빈 칸을 채우기 위해 누구의 글을 기다리지 않는다.
대신 디스어셈블을 열거나 트레이스를 건다.

여전히 안 보이는 것은 분명히 있다. 그게 무엇인지 지금은 모른다 —
*시기 4*의 내가 알게 될 것이다.

[post-builddiff]: /posts/ntcreatefile-build-differences-ko/

## 세 시기를 한 표에

| 시기 | 도구 | 본 것 | 의미의 출처 | 한계 |
| --- | --- | --- | --- | --- |
| 대학생 | WinDbg + 심볼 | Call graph | 도구가 알려준 이름 | semantic이 비어 있음 |
| 주니어 | + Project Zero, ReactOS | 각 함수가 *무엇을 하는지* | 다른 사람의 해석 | 글이 안 다룬 박스는 존재조차 모름 |
| 현재 | + 디스어셈블, 1차 헤더, 실측 | 함수의 *행동* | 1차 자료 (본인 독해) | 알 수 없음 — 다음 시기에 알게 됨 |

## 한 가지만 기억한다면

같은 `NtCreateFile`을 세 번 봤다. 매번 새로 보인 것은 함수가 아니라
*내가 더 이상 외부 해석에 의존하지 않게 된 영역*이었다. 분석 능력의
성장은 도구가 좋아진 게 아니다. **무엇이 안 보이는지를 스스로 알아보는
능력**이 자라는 과정이었다.

이 글은 시기 4의 내가 시기 3의 나에게 무엇을 못 봤다고 말할지를
미리 예약해 두는 글이기도 하다.

---

관련 글:
- [Hyper-V vhdmp.sys 베이스라인][post1] — 측정 DNA의 다른 사례
- [vhdmp.sys 추적: WPR 세션이 VHDMP 이벤트를 0개 잡는 이유][post2] — 1차 자료 독해가 글을 구한 경우

[post1]: /posts/virtual-disk-benchmark-data-ko/
[post2]: /posts/vhdmp-etw-zero-events-ko/
