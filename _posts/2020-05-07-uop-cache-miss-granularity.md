---
title: Granularity of uops-cache miss on Skylake microarchitecture
author: Dmitrii Bundin
date: 2020-05-07 20:19:51 +0300
categories: [FrontEnd]
tags: [uop-cache]
---

##Introduction

Intel Architecture Optimization Manual (IAOM) gives somewhat vague description of the uops cache (a.k.a. Decoded ICache, DSB, Decode Stream Buffer). Starting Sandy Brindge microarchitecture the uop cache is a part of the CPU Front End and is responsible for caching microoperations of recently decoded instructions. Its primary goal is to reduce power and latency of the Front End and therefore avoid performance bottlenecks like LCP (Length Changing Prefix) which the optimization manual documents to have 3 cycles penalty.

IAOM/2.5.2.1:

> The following length changing prefixes (LCPs) imply instruction length that is different from the default
> length of instructions. Therefore they cause an additional penalty of three cycles per LCP during length
> decoding.

Basically the uop cache organization significantly differs from what we have in the standard instruction/data cache. There are 3 important aspects:

 - uops cache consists of 32 sets each of which contains 8 ways. Each way in turn can hold up to 6 micro-ops allowing up to 1536 micro-ops to be cached in total.

 - All micro-ops in a Way represent instructions which are statically contiguous in the code and have their EIPs within the same aligned 32-byte region.

 - Up to three Ways may be dedicated to the same 32-byte aligned chunk, allowing a total of 18 micro-ops to be cached per 32-byte region of the original IA program.

All of them are clearly documented in the optimization manual. The last two were copied-and-pasted directly from the IAOM/2.5.2.2.

So any time an instruction is decoded by the Legacy Decode Pipeline and is delivered to the micro-op queue it is also delivered to the uops cache. Next time the micro-op is needed it ***might*** be delivered from the uops cache bypassing Legacy Decode Pipeline. The key point here is that it might, but also it ***might not***. One possible reason for that may be that micro-ops from a 32-byte region overflow the uops cache. Intel clearly documents such case 

IAOM/B.5.7.3:

> There are no partial hits in the Decoded ICache. If any micro-op that is part of that lookup on the 32-byte
> chunk is missing, a Decoded ICache miss occurs on all micro-ops for that transaction.

There are 2 interesting facts to notice about the uops cache (IOAM/2.5.2.2): 

 - The Decoded ICache is virtually included in the Instruction cache

 - Once micro-ops are delivered from the legacy pipeline, fetching micro-ops from the Decoded ICache can resume only after the next branch micro-op

To understand what was behind the rule about requiring a branch micro-op I ran some experiments. Besides somewhat clearing the reason from that it gave some interesting results related to the uops cache miss granularity which is documented to be 32-byte as shown above.
