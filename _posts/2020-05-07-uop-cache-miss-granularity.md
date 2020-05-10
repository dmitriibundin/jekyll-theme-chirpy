---
title: Granularity of uops-cache miss on Skylake microarchitecture
author: Dmitrii Bundin
date: 2020-05-07 20:19:51 +0300
categories: [CPU, FrontEnd]
tags: [x86, uop-cache]
---

## Introduction

The [Intel Architecture Optimization Manual](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-optimization-reference-manual.html) (IAOM) provides some details about uops cache (a.k.a. Decoded ICache, DSB, Decode Stream Buffer) internal organization. Starting Sandy Brindge microarchitecture the uop cache is a part of the CPU Front End and is responsible for caching microoperations of recently decoded instructions. Its primary goal is to reduce power and latency of the Front End and therefore avoid performance bottlenecks like LCP (Length Changing Prefix) which the optimization manual documents to have 3 cycles penalty.

IAOM/2.5.2.1:

> The following length changing prefixes (LCPs) imply instruction length that is different from the default
> length of instructions. Therefore they cause an additional penalty of three cycles per LCP during length
> decoding.

Basically the uop cache organization significantly differs from what we have in the standard instruction/data cache. There are 3 important aspects to understand:

 - uops cache consists of 32 sets each of which contains 8 ways. Each way in turn can hold up to 6 micro-ops allowing up to 1536 micro-ops to be cached in total.

 - All micro-ops in a Way represent instructions which are statically contiguous in the code and have their EIPs within the same aligned 32-byte region.

 - Up to three Ways may be dedicated to the same 32-byte aligned chunk, allowing a total of 18 micro-ops to be cached per 32-byte region of the original IA program.

All of them are clearly documented in the optimization manual. The last two were copied-and-pasted directly from the IAOM/2.5.2.2.

So any time an instruction is decoded by the Legacy Decode Pipeline and delivered to the micro-op queue it is also delivered to the uops cache. Next time the micro-op is needed it ***might*** be delivered from the uops cache bypassing Legacy Decode Pipeline. The key point here is that it might, but also it ***might not***. One possible reason for that may be that micro-ops from a 32-byte region overflow the uops cache. Intel clearly documents such case 

IAOM/B.5.7.3:

> There are no partial hits in the Decoded ICache. If any micro-op that is part of that lookup on the 32-byte
> chunk is missing, a Decoded ICache miss occurs on all micro-ops for that transaction.

There are 2 interesting facts to notice about the uops cache (IAOM/2.5.2.2) though: 

 - The Decoded ICache is virtually included in the Instruction cache

 - Once micro-ops are delivered from the legacy pipeline, fetching micro-ops from the Decoded ICache can resume only after the next branch micro-op

I ran some experiments to understand what was behind the rule about requiring a branch micro-op. Besides somewhat clearing the reason it gave me some interesting *undocumented* result related to the uops cache miss granularity which is defined to be 32-bytes long as shown above.

## Drill down the uops cache

To analyze uops cache behavior we will need routines written in the Assembly language. All of the examples below are guaranteed to be assembled by NASM version 2.13.02 and run on Intel Kaby Lake i7-8550U CPU. Basically what we are going to do is to collect Performance Counters provided by the Intel PMU and try to get sensible results depending on the routine being analyzed.

The counters we are interested in are `icache_64b.iftag_hit:u`, `idq.dsb_uops:u`, `idq.mite_uops:u` and `idq.ms_uops:u`. Consider each of them separately:

 - `icache_64b.iftag_hit` - As per the Intel System Programming Manual it counts "Instruction fetch tag lookups that hit in the instruction cache (L1I)". The modern CPU caches however stores a memory address along with a cache line. The address basically consists of 3 components: offset, index and tag. The index is used to determine possible ways a line may occupy in the cache and the tag is used to check if the address is actually cached. IDK if the "Instruction fetch tag" is the same or has empty or non-empty overlapping with the address tag.

 - `idq.dsb_uops` - Represents the number of micro-ops delivered from the uops cache.

 - `idq.mite_uops` - Represents the number of micro-ops delivered from the Legacy Decode Pipeline

 - `idq.ms_uops` - Represents the number of micro-ops delivered from Microcode Sequencer

The `:u` suffix means the counters will be colleted for user code which runs in the ring 3. Basically the `:u`/`:k` suffixes instruct perf to set the appropriate bits in the `IA32_PERFEVTSEL` MSR so the counter can be later read from a corresponding `IA32_PMC` MSR. Here is how Intel System Programming Manual depicts the `IA32_PERFEVTSEL` structure:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/ia32perfevtsel_layout.png" | relative_url }})

The `USR` bit corresponds to user-space counters.

The signature of all functions we will consider have the form `void function_name(size_t iteration_count);` which will be called with `iteration_count = 1L << 31`. It will run the corresponding assembly code `iteration_count` times in a loop ending with `dec rdi`, `jnz` pair. In some examples considered below we will need to count the uops by hand in the ***fused domain***. It means the Macro Fusion applied to `dec rdi`, `jnz` will result in a signle micro-op.

Now let's go ahead and start with the simplest example to check the uops cache.

### Simple fetching from the uops cache

To understand what's going on when micro-ops are fetched from the uops cache we will need the following NASM macro and function:

```
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

The `idq.dsb_uops`, `idq.mite_uops`, `idq.ms_uops` are pretty expected. Since the 8 consecutive `nop ax` instructions consume the entire 32-bytes region all of the uops come from the uops-cache. 

The insteresting thing to notice is actually on the plot for `icache_64b.iftag_hit`. As can be seen the instruction fetch tag hit lookup happens once per 64 byte. The 64 byte in turns contains 2 32-bytes instruction regions cached in the uops cache. Considering the fact that switches from `MITE` to `DSB` require a branch micro-op to be taken it is reasonable to check how the counters are affected by the unconditional `jmp` inserted in the middle of cache lines. It yeilds to the following example.

### Fetching from the uops cache with jmp

To investigate the uops cache behavior when taking unconditional branches we need slightly more complex example in terms of NASM implementation:

```
%macro nopax7jmp 1         
    %%loop:
    %define ODD
    %rep %1
        %ifdef ODD
            times 7 nop ax
            %push
            jmp %$aligned_label
            align 32
            %$aligned_label:
            %pop
            %undef ODD
        %else
            times 8 nop ax
            %define ODD
        %endif
    %endrep
    dec rdi
    jnz %%loop
    ret
%endmacro

align 64
ucmc_64b_nopax7jmp_n:
    nopax7jmp n
```

What the macro `nopax7jmp` does is it generates blocks of assembly code. The block comprises either `times 7 nop ax` following the unconditional `jmp` to the start of the next 32-bytes aligned region or just `times 8 nop ax`. The sole macro argument specifies the total number of blocks to generate such that they alternate each other.

Running the example with perf we have the following results:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/uops-hits-jmp.png" | relative_url }})

Results depicted on the second plot meet our expectations perfectly. All of the uops are delivered from the DSB, while mite is staying idle. Moreover the result is exactly the same as we saw in the previous example. This is because `nop ax` and the near jump instruction `jmp` are both decoded into a signle micro-op.

The key difference with the previous example can be observed on the first plot. As can be seen adding `jmp`s in the middle of cache lines causes additional instruction fetch tag lookup even thought the target is located within the same cache line. Recalling the IAOM/2.5.2.2:

> Once micro-ops are delivered from the legacy pipeline, fetching micro-ops
> from the Decoded ICache can resume only after the next branch micro-op.

it made me think that MITE to DSB switches and probably the DSB lookup itself might be connected to the Instruction Fetch (IF) tag lookup. To get more insights consider an example where an L1I line contains 32-bytes chunk that do not fit the DSB.

### Overflowing DSB with too many uops

To check how the DSB behaves when micro-ops cannot fit it we can consider an example containing more then 18 consequent `nop` instructions within a 32-byte chunk. For example:

```
%macro nopax8nop19jmp 1
    %%loop:
    %rep %1
        times 8 nop ax
        times 19 nop
        %push
            jmp %$aligned_label
            align 64
            %$aligned_label:
        %pop
    %endrep
    dec rdi
    jnz %%loop
    ret
%endmacro

align 64
ucmc_64b_nopax8nop19jmp_n:
    nopax8nop19jmp n
```

What we have here is each 64-byte L1I line contains 8 `nop ax`s following 19 `nop`s ending with `jmp` to the begginning of the next 64-byte aligned chunk. The 64-byte aligned chunk is either another block or a loop conditional branch.

Before providing the results of experiments let's try to imagine what the results might be. Consider `ucmc_64b_nopax8nop19jmp_2`. Its code is kind of verbose, but I will provide it here with some comments added for clarity's sake:

```
ucmc_64b_nopax8nop19jmp_2:
    ;first 64-byte aligned block start
    nop ax
    nop ax
    nop ax
    nop ax
    nop ax
    nop ax
    nop ax
    nop ax
    ;32-bytes boundary
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    jmp 0x555555554e80 <..@100.aligned_label> 
    ;first block end
    
    ;nop's to meet 64-byte alignment
    
    ;second 64-byte aligned block start
    nop ax
    nop ax
    nop ax
    nop ax
    nop ax
    nop ax
    nop ax
    nop ax
    ;32-bytes boundary
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    nop   
    jmp 0x555555554ec0 <..@103.aligned_label> 
    ;second block end
    
    ;nop's to meet the 64-byte alignment
    
    ;the dec rdi below is 64-byte aligned
    dec rdi
    jne 0x555555554e40 <ucmc_64b_nopax8nop19jmp_2>
    ret 
```

It seems reasonably to assume that 8 `nop ax`s each 64-bytes region starts with should be delivered from DSB. The followin 19 `nop`s do not fit the DSB so they should be delivered from Legacy Decode Pipeline. Having `jmp` right after the 19 `nop`s to the next cache line and taking IAOM/2.5.5.2 "Once micro-ops are delivered from the legacy pipeline, fetching micro-ops from the Decoded ICache can resume only after the next branch micro-op" into account it seems reasonable to assume that each of the 8 `nop ax` should be delivered from DSB.

Now let's take a look what the actual results are:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/dsb-ovf-full-line-miss-iftag.png" | relative_url }})

![upload-image]({{ "/assets/img/uops-cache-miss-gran/dsb-ovf-full-line-miss-uops.png" | relative_url }})

The results for `icache_64b.iftag_hit` is pretty expected.

The intersting part of it is where uops were delivered from. As can be seen the vast majority of them were delivered from MITE. The delivery rate from DSB were constant and is approximately equal to the value of `size_t iteration_count` parameter.The only micro-op delivered from DSB on each iteration were Macro Fused `dec rdi - jnz`. All in all the result definitely does not meet our expectation. Now recall the IAOM/B.5.7.3:

> There are no partial hits in the Decoded ICache. If any micro-op that is part of that lookup on the 32-byte
> chunk is missing, a Decoded ICache miss occurs on all micro-ops for that transaction.

The example shown above suggests that the miss might not just for 32-byte lookup but for the whole cache line.
