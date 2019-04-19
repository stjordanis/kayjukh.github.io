---
title: Xilinx Zynq BareMetal cache operations
header:
  overlay_image: /assets/images/tj-holowaychuk-77921-unsplash.jpg
categories: [programming]
tags: [Xilinx, Zynq, SoC, cache, C, High Level Synthesis]
classes: wide
author_profile: true
---

Last summer, I had to perform some experiments on a Xilinx Zynq-7000 board during a research internship at IRISA, Rennes, France. Broadly speaking, my goal was to be able to fine-tune cache operations on the board by manually selecting which lines to invalidate and/or flush. I was developing in BareMetal mode, with little to no operating system support and utilities available.

There was no documented way of achieving this kind of fine control over the ARM Cortex-A9 cache on the Zynq board from a "high-level" perspective, so I first had to rely on some inline assembly. But inline assembly can quickly become ugly, and it is a real mess to maintain. I decided to search for an alternative solution. After all, there had to be a way of doing cache maintenance with little to no headaches on such a widely used platform. After some painful hours of digging into the Xilinx codebase, I stumbled upon two files with interesting names: `xil_cache.h` and `xil_cache_l.h`.

Here is the list of all function prototypes declared in the first one:
```cpp
// Data cache operations
void Xil_DCacheEnable(void);
void Xil_DCacheDisable(void);
void Xil_DCacheInvalidate(void);
void Xil_DCacheInvalidateRange(INTPTR adr, u32 len);
void Xil_DCacheFlush(void);
void Xil_DCacheFlushRange(INTPTR adr, u32 len);

// Instruction cache operations
void Xil_ICacheEnable(void);
void Xil_ICacheDisable(void);
void Xil_ICacheInvalidate(void);
void Xil_ICacheInvalidateRange(INTPTR adr, u32 len);
```

The latter header file provides a more fine-grained approach to cache maintenance by exposing functions for cache line manipulation:
```cpp
// Generic cache line operations
void Xil_DCacheInvalidateLine(u32 adr);
void Xil_DCacheFlushLine(u32 adr);
void Xil_DCacheStoreLine(u32 adr);
void Xil_ICacheInvalidateLine(u32 adr);

// L1 data cache maintenance
void Xil_L1DCacheEnable(void);
void Xil_L1DCacheDisable(void);
void Xil_L1DCacheInvalidate(void);
void Xil_L1DCacheInvalidateLine(u32 adr);
void Xil_L1DCacheInvalidateRange(u32 adr, u32 len);
void Xil_L1DCacheFlush(void);
void Xil_L1DCacheFlushLine(u32 adr);
void Xil_L1DCacheFlushRange(u32 adr, u32 len);
void Xil_L1DCacheStoreLine(u32 adr);

// L1 instruction cache maintenance
void Xil_L1ICacheEnable(void);
void Xil_L1ICacheDisable(void);
void Xil_L1ICacheInvalidate(void);
void Xil_L1ICacheInvalidateLine(u32 adr);
void Xil_L1ICacheInvalidateRange(u32 adr, u32 len);

// L2 cache maintenance
void Xil_L2CacheEnable(void);
void Xil_L2CacheDisable(void);
void Xil_L2CacheInvalidate(void);
void Xil_L2CacheInvalidateLine(u32 adr);
void Xil_L2CacheInvalidateRange(u32 adr, u32 len);
void Xil_L2CacheFlush(void);
void Xil_L2CacheFlushLine(u32 adr);
void Xil_L2CacheFlushRange(u32 adr, u32 len);
void Xil_L2CacheStoreLine(u32 adr);
```

Some further investigation--mostly in the [Xilinx Embedded Software Development](https://github.com/Xilinx/embeddedsw) GitHub repository--revealed that all of the above functions perform essentially the same task I was previously achieving with raw assembly. However, they offer the invaluable advantage of increased code readability and portability across Xilinx-compatible targets.

I have seen little to no mention of these functions in the mainline Xilinx documentation files. Some posts on the Xilinx forums mention them, but it would have been much easier if they were all listed in one place. This post humbly fulfills this wish, in the hope that it will be useful to others.
