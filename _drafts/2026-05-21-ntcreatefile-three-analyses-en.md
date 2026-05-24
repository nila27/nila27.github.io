---
title: "NtCreateFile, Analyzed Three Times"
date: 2026-05-21 10:00:00 +0900
categories: [Reflections]
tags: [ntcreatefile, kernel, filter-manager, windbg, reactos, retrospective]
lang: en
---

🇰🇷 [한국어 버전](/posts/ntcreatefile-three-analyses-ko/) ・ 🇺🇸 English (you are here)

> **TL;DR**
> - I analyzed the same `NtCreateFile` at three points in my career — undergraduate, junior, present.
> - What changed each time wasn't the function but *where I got meaning from* — tools → other people's writing → primary sources.
> - Growth in analysis wasn't about better tools. It was about no longer depending on external interpretation.
{: .prompt-tip }

`NtCreateFile` is the first syscall of the Windows file system, and the
entry point for opening every file, device, named pipe, and mailslot.
Analyses of this function are already abundant online. This post isn't
trying to add another such analysis — it records *how I saw the same
function differently across three points in time*.

When you look at the same target through time, what becomes newly
visible isn't the function. It's *what the earlier you didn't see*.

## Stage 1 — Undergraduate: names the symbols give you

Tools were WinDbg with Microsoft's public symbol server. I'd set a
breakpoint on `NtCreateFile` in `ntdll.dll`, take call stacks with `kb`,
step into the calls, and write down *which functions were called* on
paper.

The diagram I drew looked roughly like this:

```
NtCreateFile
└─ IoCreateFile (or IoCreateFileEx)
   └─ ObOpenObjectByName (...)
      └─ ObpLookupObjectName (...)
         └─ IopParseDevice (...)
            └─ IofCallDriver → IRP_MJ_CREATE → (driver routine)
```
{: file='notes-undergrad.txt' }

This diagram *wasn't wrong*. The call relationships really are this.
But this diagram was *all I saw* at that time. What I wrote next to
each function is the key — **I wrote only the function names**. I
didn't know *what* `ObpLookupObjectName` *does*, and I didn't know
where to look for primary sources to find out.

Symbols tell you that a function exists. They don't tell you what it
does. I didn't yet recognize the boundary between questions a tool can
answer and questions it can't.

The limit of this stage was simple — **I saw the call graph, but the
semantics were empty.**

## Stage 2 — Junior: someone else's writing filled in the meaning

Two things arrived at the same time.

One was [James Forshaw's post on Project Zero][forshaw1]. Treating
`NtCreateFile` from a *security angle* — `PreviousMode`, `AccessMode`,
*when* captured arguments are validated — the post filled in, for the
first time, *what input the function trusts*, next to the box on my
paper that had just read "`ObpLookupObjectName`".

The other was the [ReactOS source][reactos]. An open-source
reimplementation of the NT kernel isn't the official Windows source,
but it let me read *code with the same intent*. The parts Forshaw's
post described as "this is what it means", I could verify in ReactOS's
`IoCreateFile` / `ObpLookupObjectName` as *the corresponding code*.

With these two materials side by side, the diagram I drew next had a
one-line annotation by each function name:

```
NtCreateFile
└─ IoCreateFile
   ├─ AccessMode/PreviousMode capture       ← decides who's calling
   ├─ OBJECT_ATTRIBUTES capture/validate    ← safe copy from user to kernel
   └─ ObOpenObjectByName
      └─ ObpLookupObjectName
         ├─ namespace traversal             ← \??\, \Device, ...
         ├─ symbolic link / reparse         ← can re-enter the same syscall
         └─ IopParseDevice
            ├─ FILE_OBJECT allocation
            ├─ Security check               ← in which mode?  ← Forshaw's core point
            └─ IRP_MJ_CREATE → driver
```
{: file='notes-junior.txt' }

The only difference from the undergraduate version was that a one-line
description of *what each function does* now sat next to its name. The
call graph was the same. What had been filled in was the semantics.

The real skill at this stage was **reading**. The ability to read what
someone else wrote and what someone else implemented, and *transfer it
into my own notes*. Dependent but powerful — without this stage, the
next one is impossible.

There were still things I couldn't see. Parts Forshaw's post didn't
cover — the ECP (Extra Create Parameters) chain, where exactly the
Filter Manager hooks in, how oplocks and transactions interact —
remained as blank space in my notes. I didn't even know the blanks
were there. *Without someone else writing about it, I wouldn't even
know the box existed.*

[forshaw1]: https://googleprojectzero.blogspot.com/2019/03/windows-kernel-logic-bug-class-access.html
[reactos]: https://github.com/reactos/reactos

## Stage 3 — Present: from primary sources, directly

At this stage the kind of material I depend on changed. More precisely,
the move was *from interpreted sources to primary sources*.

- **Disassembly**: I read `ntoskrnl.exe` and `fltmgr.sys` directly.
  Symbols tell you that a function exists; disassembly tells you the
  function's *behavior*. The tool from Stage 1 is now used with the
  skill from Stage 2.
- **Public headers and official docs**: `wdm.h`, `ntifs.h`,
  `fltKernel.h`. Intent is embedded in the comments.
- **Live measurement**: I confirm what disassembly *says* with actual
  traces. ETW, WinDbg conditional breakpoints, `!fileobj`, `!devstack`.

The value of these tools shows up most sharply when *following a
moving target*. Forshaw's post is a snapshot of the build it was
written against. ReactOS is a reimplementation aimed at a particular
NT version. Both are frozen photographs in time — there's no
guarantee *the build I'm looking at* is the same as either photo.

One concrete case of Stage 3 in action is its own post —
[NtCreateFile behaves differently per build — two cases][post-builddiff].
That post shows what Stage 3 actually does, in code.

The core of this stage is *awareness* more than *skill*. I recognize
on my own where the blanks in my notes are. I don't wait for someone
to write about them. I open the disassembler or set up a trace.

There are still things I can't see. I don't know what they are right
now — *Stage 4 me* will.

[post-builddiff]: /posts/ntcreatefile-build-differences-ko/

## Three stages in one table

| Stage | Tools | What I saw | Source of meaning | Limit |
| --- | --- | --- | --- | --- |
| Undergraduate | WinDbg + symbols | Call graph | Names the tool gave | Empty semantics |
| Junior | + Project Zero, ReactOS | What each function *does* | Someone else's interpretation | Boxes others didn't write about, I didn't know existed |
| Present | + disassembly, primary headers, measurement | The function's *behavior* | Primary sources (my own reading) | Unknown — I'll know in the next stage |

## If you remember one thing

I looked at the same `NtCreateFile` three times. Each time, what was
newly visible wasn't the function — it was *the area where I no longer
needed someone else's interpretation*. Growth in analysis isn't about
better tools. It's the slow process of **learning to see what you
can't see**.

This post is also a reservation — for Stage 4 me to tell Stage 3 me
what wasn't visible.

---

Related:

- [Hyper-V vhdmp.sys baseline][post1] — another instance of the measurement DNA
- [Tracing vhdmp.sys: why your WPR session captures zero VHDMP events][post2] — a case where reading primary sources saved the post

[post1]: /posts/virtual-disk-benchmark-data-en/
[post2]: /posts/vhdmp-etw-zero-events-en/
