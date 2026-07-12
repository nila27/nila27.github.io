---
title: "Windows NPU Driver — Traces of Development"
date: 2026-07-12 10:00:00 +0900
categories: [Hardware, NPU]
tags: [npu, index]
lang: en
---

🇰🇷 [한국어 버전](/posts/npu-driver-journey-ko/) ・ 🇺🇸 English (you are here)

> **TL;DR**
> - I studied KMDF to prepare for applying to an NPU company.
> - No real hardware, so I first faked a BAR, then built an NPU device in QEMU and rewrote the driver as a real PCI function driver.
> - QEMU turned out too slow to iterate on. Work is paused. Detailed posts get linked from this page as they land.
{: .prompt-tip }

I picked up KMDF to apply to an NPU company. My home PC runs Windows 10
Home and has no NPU.

## 1. Faking a BAR

I learned KMDF by following the YouTube lectures by Bongseok Lee, CEO of
Haje Soft. One software driver grabs physical memory with
`MmAllocateContiguousMemory`, and another driver maps that physical
address as if it were a BAR.

*→ Detailed post coming.*

## 2. Building an NPU device in QEMU

I attached a fake NPU PCI device to QEMU — a handful of registers, one
MSI, one doorbell. The driver was rewritten as a real PCI function
driver. The spec belongs in a separate post.

The device evolved through a few versions after that. Each version's
spec, driver changes, and gotchas will get their own post.

*→ Detailed post coming.*

## 3. Why it's paused — QEMU is slow

The boot-QEMU-then-fire-an-IOCTL cycle became too slow to iterate on.
I built a software simulator (same IOCTL contract, implemented on the
CPU) as a workaround, but it doesn't exercise the register / MSI / DMA
protocol. Work is paused here.

*→ Detailed post coming.*

---

*P.S. If a personally-purchasable NPU existed, none of this workaround would have been necessary.*
