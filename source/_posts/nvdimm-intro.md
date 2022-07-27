---
title: nvdimm 技术与编程模型概览
date: 2022-07-18 16:45:16
updated: 2022-07-27 16:53:05
cover: https://s2.loli.net/2022/07/27/qNvb2iJmUzGPH1t.png
thumbnail: https://s2.loli.net/2022/07/27/qNvb2iJmUzGPH1t.png
categories:
- 硬件
tags:
- 硬件
- nvdimm
toc: true
---

nvdimm，即非易失性双列直插式内存模块（non-volatile DIMM），相对于传统的易失性内存，nvdimm 在断电后其中的内容也不会消失。

<!-- more -->

定义于 ACPI（NFIT），UEFI（BTT），带外通信使用 DSM。

交错本来是传统易失性内存中的一个概念。N 代表数字，和 N 通道不是一个概念，只有交错后才能叫做 N 路.

### PMEM

#### DAX

#### BTT

### 实现

Linux 上关于持久内存的配置概念方面曾经有过一些变动，本章节将以本文撰写时最新内核和工具的支持情况为基础。
