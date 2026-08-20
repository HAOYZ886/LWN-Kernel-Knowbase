---
title: "Grand Schemozzle: Spectre continues to haunt"
url: https://lwn.net/Articles/795637/
date: "August 8, 2019"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet August 8, 2019"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 8, 2019

The [Spectre v1](<https://lwn.net/Articles/742702/>) hardware vulnerability is often characterized as allowing array bounds checks to be bypassed via speculative execution. While that is true, it is not the full extent of the shenanigans allowed by this particular class of vulnerabilities. For a demonstration of that fact, one need look no further than the "`SWAPGS` vulnerability" known as CVE-2019-1125 to the wider world or as "Grand Schemozzle" to the select group of developers who addressed it in the Linux kernel. 

Segments are mostly an architectural relic from the earliest days of x86; to a great extent, they did not survive into the 64-bit era. That said, a few segments still exist for specific tasks; these include `FS` and `GS`. The most common use for `GS` in current Linux systems is for thread-local or CPU-local storage; in the kernel, the `GS` segment points into the per-CPU data area. User space is allowed to make its own use of `GS`; the [`arch_prctl()`](<http://man7.org/linux/man-pages/man2/arch_prctl.2.html>) system call can be used to change its value. 

As one might expect, the kernel needs to take care to use its own `GS` pointer rather than something that user space came up with. The x86 architecture obligingly provides an instruction, `SWAPGS`, to make that relatively easy. On entry into the kernel, a `SWAPGS` instruction will exchange the current `GS` segment pointer with a known value (which is kept in a [model-specific register](<https://en.wikipedia.org/wiki/Model-specific_register>)); executing `SWAPGS` again before returning to user space will restore the user-space value. Some carefully placed `SWAPGS` instructions will thus prevent the kernel from ever running with anything other than its own `GS` pointer. Or so one would think. 

There is a slight catch, in that not every entry into kernel code originates from user space. Running `SWAPGS` if the system is already running in kernel mode will not lead to good things, so the actual code in the kernel in most cases is the assembly equivalent of: 

```
if (!in_kernel_mode())
        	SWAPGS
```

That, of course, is where things can go wrong. If that code is executed speculatively, the processor may make the wrong decision about whether to execute `SWAPGS` and run with the wrong `GS` segment pointer. This test can be incorrectly speculated either way. If the CPU is speculatively executing an entry from user space, it may decide to forego `SWAPGS` and run with the user-space `GS` value. If, instead, the system was already running in kernel mode, the CPU might again speculate incorrectly and execute `SWAPGS` when it shouldn't, causing it to run (speculatively) with a user-space `GS` value. Either way, subsequent per-CPU data references would be redirected speculatively to an address under user-space control; that enables data exfiltration by way of the usual side-channel techniques. 

That looks like a wide-open channel into kernel data structures, but there are some limitations. Only Intel processors will execute `SWAPGS` speculatively, so the already-in-kernel-mode case is limited to those processors. When entering from user mode, though, the lack of a needed `SWAPGS` instruction can obviously be speculated on any processor. 

The other roadblock for attackers is that, while `arch_prctl()` can be used by unprivileged code to set the `GS` pointer, it limits that pointer to user-space addresses. That does not entirely head off exploitation, but it makes it harder: an attacker must find kernel code that loads a value via `GS`, then uses that value as a pointer that is dereferenced in turn. As Josh Poimboeuf notes in [the mitigation patch](<https://git.kernel.org/linus/a2059825986a>) merged into the mainline: 

It's difficult to audit for this gadget in all the handlers, so while there are no known instances of it, it's entirely possible that it exists somewhere (or could be introduced in the future). Without tooling to analyze all such code paths, consider it vulnerable. 

The use of [supervisor mode access prevention](<https://lwn.net/Articles/517475/>) will block this attack — but only on processors that are not vulnerable to the Meltdown problem, so that is only so helpful. 

It is also worth noting that there is a longstanding effort to [add support for the `FSBASE` and `GSBASE` instructions](<https://lwn.net/Articles/787860/>), which allow direct (and uncontrolled) setting of `GS` from user space. There are a number of performance advantages to allowing this, so the pressure to merge the patches is likely to continue even though they would make exploiting the `SWAPGS` vulnerability easier. 

The mitigation applied in the kernel is relatively straightforward: serializing (`LFENCE`) instructions are placed in the code paths that decide to (or not to) execute `SWAPGS`. This, of course, will slow execution down, which is why [the pull request](<https://git.kernel.org/linus/4368c4bc9d36>) for these fixes described them as coming from ""the performance deterioration department"". On systems where these attacks are not a concern, the new barriers can be disabled (along with all other Spectre v1 defenses) with the `nospectre_v1` command-line option. 

The Spectre vulnerabilities were so-named because it was assumed that they would haunt us for a long time. The combination of speculative execution and side channels leads to a huge variety of possible attacks and an equally large difficulty in proving that no such attacks are possible in any given body of code. As a result, the pattern we see here — slowing down the system to defend against attacks that may or may not be practical — is likely to be with us for some time yet.
