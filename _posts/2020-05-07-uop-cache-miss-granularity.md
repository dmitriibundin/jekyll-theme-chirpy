---
title: Granularity of uops-cache miss on Skylake microarchitecture
author: Dmitrii Bundin
date: 2020-05-07 20:19:51 +0300
categories: [CPU, FrontEnd]
tags: [uop-cache]
---

## Introduction

[Intel Architecture Optimization Manual](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-optimization-reference-manual.html) (IAOM) gives somewhat vague description of the uops cache (a.k.a. Decoded ICache, DSB, Decode Stream Buffer). Starting Sandy Brindge microarchitecture the uop cache is a part of the CPU Front End and is responsible for caching microoperations of recently decoded instructions. Its primary goal is to reduce power and latency of the Front End and therefore avoid performance bottlenecks like LCP (Length Changing Prefix) which the optimization manual documents to have 3 cycles penalty.

IAOM/2.5.2.1:

> The following length changing prefixes (LCPs) imply instruction length that is different from the default
> length of instructions. Therefore they cause an additional penalty of three cycles per LCP during length
> decoding.

Basically the uop cache organization significantly differs from what we have in the standard instruction/data cache. There are 3 important aspects to understand:

 - uops cache consists of 32 sets each of which contains 8 ways. Each way in turn can hold up to 6 micro-ops allowing up to 1536 micro-ops to be cached in total.

 - All micro-ops in a Way represent instructions which are statically contiguous in the code and have their EIPs within the same aligned 32-byte region.

 - Up to three Ways may be dedicated to the same 32-byte aligned chunk, allowing a total of 18 micro-ops to be cached per 32-byte region of the original IA program.

All of them are clearly documented in the optimization manual. The last two were copied-and-pasted directly from the IAOM/2.5.2.2.

So any time an instruction is decoded by the Legacy Decode Pipeline and is delivered to the micro-op queue it is also delivered to the uops cache. Next time the micro-op is needed it ***might*** be delivered from the uops cache bypassing Legacy Decode Pipeline. The key point here is that it might, but also it ***might not***. One possible reason for that may be that micro-ops from a 32-byte region overflow the uops cache. Intel clearly documents such case 

IAOM/B.5.7.3:

> There are no partial hits in the Decoded ICache. If any micro-op that is part of that lookup on the 32-byte
> chunk is missing, a Decoded ICache miss occurs on all micro-ops for that transaction.

There are 2 interesting facts to notice about the uops cache (IAOM/2.5.2.2): 

 - The Decoded ICache is virtually included in the Instruction cache

 - Once micro-ops are delivered from the legacy pipeline, fetching micro-ops from the Decoded ICache can resume only after the next branch micro-op

To understand what was behind the rule about requiring a branch micro-op I ran some experiments. Besides somewhat clearing the reason it gave me some interesting results related to the uops cache miss granularity which is documented to be 32-byte as shown above.

## Drill down the uops cache

To analyze uops cache behavior we will need routines written in the Assembly language. All of the examples below are guaranteed to be assembled by NASM version 2.13.02 and run on Intel Kaby Lake i7-8550U CPU. Basically what we are going to do is to collect Performance Counters provided by the Intel PMU and try to get sensible results depending on the routine being analyzed.

The counters we are interested in are `icache_64b.iftag_hit:u`, `idq.dsb_uops:u`, `idq.mite_uops:u` and `idq.ms_uops:u`. Consider each of them separately:

 - `icache_64b.iftag_hit` - This counters collect cache tag hits. The modern CPU caches stores a memory address along with the cache line. The address basically consists of 3 components: offset, index and tag. Index is used to determine possible positions the line may occupy in the cache and tag is used to check if the address is actually cached. The counter represents the amount of L1I tag lookup hits.

 - `idq.dsb_uops` - Represents the number of micro-ops delivered from the uops cache.

 - `idq.mite_uops` - Represents the number of micro-ops delivered from the Legacy Decode Pipeline

 - `idq.ms_uops` - Represents the number of micro-ops delivered from Microcode Sequencer

The `:u` suffix means the counters will be colleted for user code which runs in the ring 3. Basically the `:u`/`:k` suffixes instruct perf to set the appropriate bits in the `IA32_PERFEVTSEL` so the counter can be later read from `IA32_PMC`. Here is how Intel System Programming Manual depicts the `IA32_PERFEVTSEL` structure:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/ia32perfevtsel_layout.png" | relative_url }})

The `USR` bit corresponds to user-space counters.

Now let's go ahead and start with the simplest example to check the uops cache.

### Simple fetching from the uops cache

To understand what's going on when micro-ops are fetched from the uops cache we will need the following NASM macro and function:

```Asm
    %macro nopax8_rep 1        
        %%loop:
        %rep %1                
             times 8 nop ax
        %endrep
        dec rdi
        jnz %%loop
        ret
    %endmacro
 
    align 64
    ;n in the function name is the number of
    ;times we want to repeat 'times 8 nop ax'.
    ucmc_64b_nopax8_n:  
        nopax8_rep n
```

We have the following results:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/simple-uops-hit.png" | relative_url }})

The `idq.dsb_uops`, `idq.mite_uops`, `idq.ms_uops` are pretty expected. Since the 8 consecutive `nop ax` instructions consume the entire 32-bytes chunk all of the uops come from the uops-cache. 

The insteresting thing to notice is actually on the plot for `icache_64b.iftag_hit`. As can be seen the instruction cache tag hit lookup happens once per 64 byte. The 64 byte in turns contains 2 32-bytes instruction regions cached in the uops cache. Considering the fact that switches from `MITE` to `DSB` require a branch micro-op to be taken it is reasonable to check how the counters are affected by the unconditional `jmp` inserted in the middle of cache lines. It yeilds to the following example.

### Fetching from the uops cache with jmp
