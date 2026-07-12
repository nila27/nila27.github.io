---
title: "Windows NPU Driver로 개발 흔적"
date: 2026-07-12 10:00:00 +0900
categories: [Hardware, NPU]
tags: [npu, index]
lang: ko-KR
---

> **3줄 요약**
> - NPU 회사에 가기 위한 준비로 KMDF 드라이버를 공부했다.
> - 실 하드웨어가 없어 처음엔 BAR를 흉내내 실험, 이후 QEMU에 NPU 디바이스를 만들어 실 PCI 드라이버로 옮겨갔다.
> - QEMU가 너무 느려 지금은 멈춘 상태. 세부 글은 이 페이지에서 하나씩 링크한다.
{: .prompt-tip }

🇰🇷 한국어 (현재) ・ 🇺🇸 [English version](/posts/npu-driver-journey-en/)

NPU 회사에 지원하려고 KMDF를 공부 했었다. 집 PC는 Windows 10 Home, NPU device가 없어서 한계가 있었다.

## 1. BAR를 흉내내던 시절

KMDF는 하제소프트 이봉석 대표 유튜브 강의를 보며 간단히 따라 만들었다.
소프트웨어 드라이버 하나가 `MmAllocateContiguousMemory`로 물리 메모리를
잡고, 다른 드라이버가 그 물리주소를 BAR처럼 매핑해 쓰는 구성.

*→ 세부 글 준비 중.*

## 2. QEMU에 NPU 디바이스를 만든 시절

QEMU에 가짜 NPU PCI 디바이스 하나를 붙였다 — 레지스터 몇 개, MSI 1개,
도어벨 하나. 드라이버는 실 PCI 펑션 드라이버로 다시 썼다. 스펙은
별도 글로 다룬다.

이후 디바이스는 몇 번 진화했다. 각 버전의 사양·드라이버 변화·만난
문제는 별도 글로 다룬다.

*→ 세부 글 준비 중.*

## 3. 지금은 멈춘 이유 — QEMU가 느림

QEMU 부팅 후 IOCTL 하나 던져보는 사이클이 느려 반복이 실용적이지
않았다. 같은 IOCTL 계약을 CPU로 구현한 SW 시뮬레이터를 따로 뒀지만
레지스터·MSI·DMA 프로토콜은 검증되지 않는다. 진행은 여기서 멈췄다.

*→ 세부 글 준비 중.*

---

*P.S. 개인이 살 수 있는 NPU 하나만 있었으면 이 우회 전부 필요 없었다.*
