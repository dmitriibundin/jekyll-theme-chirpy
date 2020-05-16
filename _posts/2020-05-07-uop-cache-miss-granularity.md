---
title: Undocumented miss granularity of the uops cache on Skylake microarchitecture.
author: Dmitrii Bundin
date: 2020-05-16 04:30:46 +0300
categories: [CPU, FrontEnd]
tags: [x86, uops-cache]
---

## Introduction

Starting Sandy Brindge microarchitecture the uops cache (a.k.a. Decoded ICache, DSB, Decode Stream Buffer) is a part of the CPU Front End and is responsible for caching microoperations of recently decoded instructions. Its primary goal is to reduce power and latency of the Front End and therefore avoid performance bottlenecks like LCP (Length Changing Prefix) which the [Intel Architecture Optimization Manual](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-optimization-reference-manual.html) (IAOM) documents to have 3 cycles penalty.

IAOM/2.5.2.1:

> The following length changing prefixes (LCPs) imply instruction length that is different from the default
> length of instructions. Therefore they cause an additional penalty of three cycles per LCP during length
> decoding.

The uops cache structure significantly differs from what we have in the standard instruction/data cache. IAOM provides some details about its internal organization and there are 3 important aspects to understand:

 - uops cache consists of 32 sets each of which contains 8 ways. Each way in turn can hold up to 6 micro-ops allowing up to 1536 micro-ops to be cached in total.

 - All micro-ops in a Way represent instructions which are statically contiguous in the code and have their EIPs within the same aligned 32-byte region.

 - Up to three Ways may be dedicated to the same 32-byte aligned chunk, allowing a total of 18 micro-ops to be cached per 32-byte region of the original IA program.

Any time an instruction is decoded by the Legacy Decode Pipeline and delivered to the micro-op queue it is also delivered to the uops cache. Next time the micro-op is needed it ***might*** be delivered from the uops cache bypassing the Legacy Decode Pipeline. The key point here is that it might, but also it ***might not***. One possible reason for that may be the micro-ops belonging to a 32-byte region overflow the uops cache. Intel clearly documents such case 

IAOM/B.5.7.3:

> There are no partial hits in the Decoded ICache. If any micro-op that is part of that lookup on the 32-byte
> chunk is missing, a Decoded ICache miss occurs on all micro-ops for that transaction.

Basically what we are going to do during this article is to check how the statement above works on real examples. 

Before starting out the research there are 2 more important facts to understand about the uops cache

IAOM/2.5.2.2: 

 - The Decoded ICache is virtually included in the Instruction cache

 - Once micro-ops are delivered from the legacy pipeline, fetching micro-ops from the Decoded ICache can resume only after the next branch micro-op

What these 2 rules suggest us to check is how (predicted to be taken) branches affect the microarchitectural state of the Front End when fetching from the uops cache and how the uops cache interacts with L1I.

Let's go ahead and get started.

## Drill down the uops cache

To analyze the uops cache behavior we will need routines written in the Assembly language. All of the examples below are guaranteed to be assembled by NASM version 2.13.02 and run on Intel Kaby Lake i7-8550U CPU. Basically what we are going to do is to collect Performance Counters provided by the Intel PMU and try to come to a sensible conclusion depending on what a particular routine does.

The counters we are interested in are `icache_64b.iftag_hit:u`, `idq.dsb_uops:u`, `idq.mite_uops:u` and `idq.ms_uops:u`. Consider each of them separately:

 - `icache_64b.iftag_hit` - As per the [Intel System Programming Manual](https://software.intel.com/content/dam/develop/public/us/en/documents/325384-sdm-vol-3abcd.pdf) it counts "Instruction fetch tag lookups that hit in the instruction cache (L1I)". The modern CPU caches however stores a memory address along with a cache line. The address basically consists of 3 components: offset, index and tag. The index is used to determine possible ways a line may occupy in the cache and the tag is used to check if the address is actually cached. IDK if the "Instruction fetch tag" is the same or has empty or non-empty overlapping with the address tag.

 - `idq.dsb_uops` - Counts the number of micro-ops delivered from the uops cache.

 - `idq.mite_uops` - Counts the number of micro-ops delivered from the Legacy Decode Pipeline

 - `idq.ms_uops` - Counts the number of micro-ops delivered from Microcode Sequencer

The `:u` suffix means that the counters will be collected for user code only which runs in the ring 3. It is natively supported by the Intel hardware and implemented as a dedicated bit in the `IA32_PERFEVTSEL` MSR. A counter programmed with one of the `IA32_PERFEVTSEL`s can later be read from the corresponding `IA32_PMC` MSR. Here is how the Intel System Programming Manual depicts the `IA32_PERFEVTSEL` structure:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/ia32perfevtsel_layout.png" | relative_url }})

The `USR` bit corresponds to user-space counters.

Most of the examples used in this article will be implemented as NASM macros which basically consist of a loop with iteration count specified in the `rdi` register. The iteration count used in the examples will be set to `1 << 31` which seems optimal in the sense of collecting counters and time spent on running the examples. Here is how the main program entry point look like:

```
ITERATION_COUNT equ 1 << 31

%macro sys_exit 1
    mov eax, 0x3C
    mov edi, %1
    syscall
%endmacro

section .text

global _start

_start:
    mov rdi, ITERATION_COUNT
    ;the macro to measure
    sys_exit 0
```

<details> 
  <summary>About the assembly implementation of the main entry point</summary>
  Initially it was implemented in C and planned to be rewritten in C++ to automate code generation using templates, but after a while it turned out to be impractical and add unnecessary complications when communicating to hand-written assembly code.
</details>

In some examples considered below we will need to count the uops by hand which is done in the ***fused domain***. It means that Macro Fused pair `dec rdi`, `jnz` will be accounted as a signle micro-op.

The source code of all the examples in this article is available on my [GitHub repository](https://github.com/dmitriibundin/microarch-check/tree/master/ucache-miss-granularity). If you find any mistake, leaving a comment here or openinig a pull request is highly appreciated. Talk less, work more, and here is the first example:

### Simple fetching from the uops cache

To understand what's going on when micro-ops are fetched from the uops cache we will need the following NASM macro:

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
```

The macro argument specifies the number of times to repeat `times 8 nop ax`. We have the following results:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/simple-uops-hit.png" | relative_url }})

The `idq.dsb_uops`, `idq.mite_uops`, `idq.ms_uops` are pretty expected. All of the uops are delivered from the uops cache since 8 consecutive `nop ax` instructions consume the entire 32-bytes region. 

The insteresting thing to notice is actually on the plot of `icache_64b.iftag_hit`. As can be seen the instruction fetch tag hit lookup happens once per 64 bytes. The 64 bytes in turns contains 2 32-bytes instruction regions cached in the uops cache. Considering the fact that switches from `MITE` to `DSB` require a branch micro-op to be taken it is reasonable to check how the counters are affected by the unconditional `jmp` inserted in the middle of cache lines. This brings us to the following example:

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
```

What the macro `nopax7jmp` does is it generates blocks of assembly code. Each block comprises either `times 7 nop ax` following the unconditional `jmp` to the start of the next 32-bytes aligned region or just `times 8 nop ax`. The sole macro argument specifies the total number of such blocks to generate so that they alternate each other.

Running the example with perf we have the following results:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/uops-hits-jmp.png" | relative_url }})

Results depicted on the second plot meet our expectations perfectly. All of the uops are delivered from the DSB, while MITE is staying idle. Moreover the result is exactly the same as we saw on the previous example. This is because `nop ax` and the near jump instruction `jmp` are both decoded into a signle micro-op.

The key difference with the previous example can be observed on the first plot. As can be seen adding `jmp`s in the middle of cache lines causes additional instruction fetch tag lookup even thought the target is located within the same cache line. Recalling IAOM/2.5.2.2:

> Once micro-ops are delivered from the legacy pipeline, fetching micro-ops
> from the Decoded ICache can resume only after the next branch micro-op.

It might mean that MITE to DSB switches and probably the DSB lookup itself is connected to the Instruction Fetch (IF) tag lookup. To get more insights consider an example where an L1I line contains 32-bytes chunk that do not fit the DSB.

### Overflowing DSB with too many uops

To check how the DSB behaves when micro-ops cannot fit it we can consider an example containing more then 18 consecutive `nop` instructions within a 32-byte chunk. Here it is:

```
%macro nopax8nop19jmp 1
    %assign iteration_count %1
    align 64
    %%loop:
    %rep %1
        times 8 nop ax
        times 19 nop
        %assign iteration_count iteration_count-1
        %if iteration_count > 0
            %push
                jmp %$aligned_label
                align 64
                %$aligned_label:
            %pop
       %else
            dec rdi
            jnz %%loop
       %endif
    %endrep
%endmacro
```

What we have here is each 64-byte L1I line contains 8 `nop ax`s following 19 `nop`s ending with either `jmp` to the begginning of the next 64-byte aligned chunk or a loop conditional branch.

Before providing the results of the experiments let's try to imagine how it might look like. For specificity consider the `nopax8nop19jmp 2` macro invokation. Its code is kind of verbose, but I will provide it here with some comments added for clarity's sake:

```
;first 64-byte aligned block start
<..@2.loop> nop    ax
nop    ax
nop    ax
nop    ax
nop    ax
nop    ax
nop    ax
nop    ax
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
jmp    0x400100 <..@5.aligned_label> ;jump to the second block start
;first block end

;nop's to meet 64-byte alignment

;second 64-byte aligned block start
<..@5.aligned_label> nop    ax
nop    ax                                                                                                                                                             
nop    ax                                                                                                                                                             
nop    ax                                                                                                                                                             
nop    ax                                                                                                                                                             
nop    ax                                                                                                                                                             
nop    ax                                                                                                                                                             
nop    ax                                                                                                                                                             
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
dec    rdi                                                                                                                                                            
jne    0x4000c0 <..@2.loop> 
;second block end
```

The 19 `nop`s at the end of each cache line do not fit the DSB so the switching to the Legacy Decode Pipeline should occur and all of the micro-ops should be delivered from there. Having `jmp` to the next cache line right after the 19 `nop`s and taking IAOM/2.5.5.2 

>Once micro-ops are delivered from the legacy pipeline, fetching micro-ops from the Decoded ICache can resume only after the next branch micro-op

into account it seems reasonable to assume that all of the 8 `nop ax`s each 64-bytes region starts with should be delivered from the DSB.

Now let's take a look at what the actual results are:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/dsb-ovf-full-line-miss-iftag.png" | relative_url }})

![upload-image]({{ "/assets/img/uops-cache-miss-gran/dsb-ovf-full-line-miss-uops.png" | relative_url }})

The result does not meer our expectation. The most ineteresting thing here is that the number of uops delivered from the DSB were exactly 0. In all cases. It means that all of the uops were delivered from MITE. Now recall the IAOM/B.5.7.3:

> There are no partial hits in the Decoded ICache. If any micro-op that is part of that lookup on the 32-byte
> chunk is missing, a Decoded ICache miss occurs on all micro-ops for that transaction.

The example shown above suggests that the miss might happen not just for 32-byte lookup but for the whole cache line. To better understand conditions under which such misses may occur we need to consider a few more examples.

### Partial DSB hits per 32-byte region

Now lets take a look at an example with slightly different structure that we have worked with before.

```
%macro iftag_granularity_miss 1
    align 64
    %%loop:
    times %1 nop
    jmp %%next_instruction
    %%next_instruction:
    %assign ucache_rest_space 30 - %1
    times ucache_rest_space nop
    times 32 nop
    dec rdi
    jnz %%loop
%endmacro
```

The principal thing to notice about the macro is that it contais `jmp` somewhere in the middle of the first 32-bytes region of a cache line. The `jmp` instruction itself was artificially inserted to research the impact of taken branches on the DSB misses. Basically what we have here is `nop` repeated `n` times where `n` varies between 0 and 30 inclusively. The `nop`s follow `jmp` jumping to the exactly next insruction so it does not change any control flow. After the `jmp` there there are `nop`s till the end of the cache line. At the very beginning of the next cache line there is a loop conditional branch.

Now let's take a look a the results, but at first only for `18 <= n <= 30`:


![upload-image]({{ "/assets/img/uops-cache-miss-gran/partial-miss-all-uops.png" | relative_url }})

The result is expected. The first 32-byte region clearly overflows the DSB since it contains 31 micro-ops in total. But why did I consider the result starting 18? To understand it let's take a look at the whole result set of the experiment:

![upload-image]({{ "/assets/img/uops-cache-miss-gran/partial-hits-uops.png" | relative_url }})

The result is kind of surprising, isn't it? So we clearely have a partial hit per 32-bytes lookup here. Recalling how DSB caches micro-ops per 32-bytes region the result for 17 and below might become clearer. Having 17 `nop`s and 1 `jmp` gives us 18 micro ops in total. The DSB is allowed to use at most 3 ways per 32-bytes region each of which is allowed to hold at most 6 micro op. It results in the fact that at most 18 micro-ops can be cached per 32-bytes region. And this is exactly the number we have in the experiment.

Recalling that branches predicted to be taken require Instruction Fetch Tag lookup and combining all the experiments that have been run so far I came to the following emprical observation:

>There is no partial hit per the largest region that does not require Instruction Fetch Tag lookup. It might me either cache line boundaries or branches predicted to be taken.

Unfortunately it is difficult to predict the amount of uops delivered from DSB in such cases. As was shown on the previous plot the DSB delivery rate was highly non-linear depending on the number of uops.

So to check if the observation does not break when coming to more-or-less non trivial example let's consider the following:

## Example

```
example_fun:
    align 64
    times 8 nop ax
    .loop:
    times 4 nop ax
    test edi, 0x1
    jnz .no_fetch_start
    times 2 nop ax
    .cache_line_boundary:                 ;This label indicates 64-byte boundary
    times 2 nop ax
    nop dword [eax + 1 * eax + 0x1] 
    jmp .loop_branch
    .no_fetch_start:                      ;Starting this point micro-ops are not
                                          ;expected to be fetched from the DSB 
                                          ;till the end of the cache line
    times 4 nop ax
    .32bytes_boundary:                    ;This label indicates 32-byte boundary
                                          ;within the cache line
    times 6 nop ax
    nop dword [eax + 1 * eax + 0x1] 
    jmp .loop_branch
    .loop_branch:                         ;Indicates a loop conditional branch
                                          ;Also a 64-byte boundary
    dec rdi
    jnz .loop
    ret

```

What the function does is it checks if the current value in `rdi` is even or odd and in case it is even the control flow jumps to the `.no_fetch_start` label. The region this label defines spans 2 32-bytes region within the cache line. The most import thing about it is that the second 32-bytes part is ended with the branch micro-op. There is a [jcc erratum](https://www.intel.com/content/dam/support/us/en/documents/processors/mitigations-jump-conditional-code-erratum.pdf) stating that if 32-bytes region ends with a jump micro op then the whole region misses the DSB. And that's exactly what we are going to check here. The second 32-bytes part of the `.no_fetch_start` region misses due to the erratum. Applying our emperical observation we can state that the part starting the `.no_fetch_start` label till the 32-bytes boundary will miss as well.

The expected number of MITE micro-ops would look like

```
Expected MITE uops = (1L << 31) / 2 * 12 = 12,884,901,888
```

The formula needs some explanation. The reason for division by 2 was that the control flow jumps to the `.no_fetch_start` label every 2 iterations (there was the `test edi, 0x1` instruction). Summing up all the micro-ops in the region starting from `.no_fetch_start` we have 

```
times 4 nop ax +
times 6 nop ax +
nop dword [eax + 1 * eax + 0x1] +
jmp .loop_branch = 4 + 6 + 1 + 1 = 12
```

Recalling that the iteration count was defined to be `1L << 31` we have exactly the formula above.

Now let's run it under perf event and take a look at the actual results:

```
Performance counter stats for './bin':

     6 435 798 367      icache_64b.iftag_hit:u
    19 186 853 013      idq.dsb_uops:u
    12 988 209 611      idq.mite_uops:u
     9 751 861 923      cycles

       2,462670271 seconds time elapsed

       2,462579000 seconds user
       0,000000000 seconds sys
```

Our expectation is very close to the result reported by perf events: `12 884 901 888` vs `12 988 209 611`.

## Conclusion

The uops cache is clearly documented in the Intel Software Optimization Manuals, but there are nuances:

 - At least on **Kaby Lake i7-8550U** uops cannot be delivered from the DSB if any of them cannot fit it within the Instruction Fetch Tag boundary. The boundary may be defined with either a branch micro-op or a cache line end.

 - Even though uops may hit the DSB within the IF Tag boundary it is difficult to predict how many of them will actually be delivered from it (See [the example above](#partial-dsb-hits-per-32-byte-region))

 - All the examples above contain no more than 1 taken branches per cache line. If there are 2 or more things get much more unpredictable so it is difficult to give a reasonable estimation even in simple cases. This is what I still how no idea about.
