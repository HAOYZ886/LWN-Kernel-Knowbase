---
title: "Blocking straight-line speculation — eventually"
url: https://lwn.net/Articles/877845/
date: "December 9, 2021"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet December 9, 2021"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 9, 2021

The Spectre class of vulnerabilities was given that name because, it was thought, these problems would haunt us for a long time. As the fourth anniversary of [the disclosure of Meltdown and Spectre](<https://lwn.net/Articles/742702/>) approaches, there is no reason to doubt the accuracy of that name. One of the more recent Spectre variants goes by the name "straight-line speculation"; it was first disclosed in June 2020, but fixes are still trying to find their way into the compilers and the kernel. 

Straight-line speculation differs somewhat from the other Spectre vulnerabilities. To review, the classic Spectre problem afflicts code like this: 

```
if (offset < obj->array_length)
           do_something_with(obj->array[offset]);
```

On its face, this code is safe; it will only attempt to index into `obj->array` if the given offset is within bounds. A CPU running this code, though, may be unable to fetch `obj->array_length` from cache, meaning that it will have to wait for that value to come from memory. Rather than do nothing, the CPU can make a guess as to how the comparison will turn out and continue execution in a speculative mode; it may guess wrong and index `obj->array` with an out-of-bounds `offset`. Again, this shouldn't be a problem; once the array length shows up and it becomes clear that the branch was not correctly predicted, the speculative work will be thrown away. 

The problem, of course, is that this speculative execution can leave traces elsewhere in the system (most often the memory caches) that can be used to exfiltrate data that an attacker would otherwise be unable to access. In the worst cases, Spectre vulnerabilities can be used to attack the kernel or to carry out attacks between virtual machines running on the same physical host. They are a real threat, which is why numerous mitigations have been adopted to thwart these attacks despite a high performance cost. 

Straight-line speculation, which was initially disclosed in [this white paper from Arm](<https://developer.arm.com/support/arm-security-updates/speculative-processor-vulnerability/downloads/straight-line-speculation>), differs in that it does not depend on erroneous branch prediction; indeed, no conditional branches are involved at all. Instead, it takes advantage of some strange behavior around _unconditional_ control-flow changes. There are a lot of instructions that will result in a change to the program counter; on Arm, these include instructions that generate exceptions, but also unconditional direct branches and the `RET` instruction to return from a function call. 

One would think that it would not be all that hard for a speculative-execution mode to do the right thing (stop speculation, if nothing else) when it encounters one of these instructions. But, as it turns out, on some CPUs, in some situations, the CPU can execute speculatively right past one of those instructions as if it were not there — in a straight line, as it were. If an attacker can place code that will carry out the desired access immediately after one of these instructions, that code may be executed speculatively, with the usual effects elsewhere in the system. It is a hard vulnerability to exploit (it's not clear that anybody has done it in the real world), but it is still a vulnerability that is worth fixing. 

This vulnerability in Arm CPUs was assigned [CVE-2020-13844](<https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-13844>). Patches were quickly added to GCC and LLVM to address the problem; the new `-mharden-sls` flag can be used to turn the new mitigations on. Happily, this problem is relatively easy to fix; placing a barrier instruction after the unconditional jumps will do the trick. Since the jumps are, as noted, unconditional, those barriers will not actually be executed for real, but they will be encountered during straight-line speculative execution, at which point the speculation will stop. So there is no real performance impact, other than a slight increase in executable size, from this change. 

As it happens, this is not just an Arm problem; some x86 processors can speculatively execute past `RET` instructions as well. The solution in this case is to place an `INT3` instruction after the `RET` to stop speculation. `INT3` is a single-byte instruction normally used by debuggers to set breakpoints; once again, it will not actually be executed when used in this way. 

The compiler changes are in place to deal with this vulnerability (though they do not yet appear in released versions), but the kernel has not yet been updated to match. Among other things, the difficulty in exploiting this vulnerability has made addressing it seem less than fully urgent. 

Jian Cai posted [a patch series](<https://lwn.net/ml/linux-kernel/20210212051500.943179-1-jiancai@google.com/>) in February adding protection for Arm, but that work has not been merged. Arm developer Will Deacon was [opposed to the patch](<https://lwn.net/ml/linux-kernel/20210305095256.GA22536@willie-the-truck/>): 

> I still don't see why SLS is worth a compiler mitigation which will affect all CPUs that run the kernel binary, but Spectre-v1 is not. In other words, the big thing missing from this is a justification as to why SLS is a problem worth working around for general C code. 

That patch series has not returned since. 

The x86 side has seen more recent work in the form of [this patch series](<https://lwn.net/ml/linux-kernel/20211204134338.760603010@infradead.org/>) from Peter Zijlstra. It, too, uses the new compiler options (when available) to insert the needed instructions, but (as Deacon had [noted](<https://lwn.net/ml/linux-kernel/20210217094859.GA3706@willie-the-truck/>) in response to the Arm patches) that is only part of the problem. There is quite a bit of assembly code in the kernel, none of which will be addressed by changes to the compilers. So Zijlstra laboriously went through that code, replacing all `RET` instructions with a macro that can, when the right configuration options are selected, expand to include the `INT3` instruction as well. For good measure, the `objtool` utility is also enhanced to check that `INT3` instructions have been inserted where needed as a way of preventing potential vulnerabilities from being added in the future. 

Given that nearly a year and a half has passed since this vulnerability was disclosed, it is clear that nobody is feeling that it must be addressed urgently. The slow progress toward protection from straight-line speculation is a far cry from the scrambling that took place prior to the initial Meltdown and Spectre disclosures. But one thing we should have learned by now is that attackers will eventually figure out a way to make use of any opening, no matter how difficult it is to exploit. So protection from straight-line speculation vulnerabilities with little performance penalty is welcome, even if it comes late.
