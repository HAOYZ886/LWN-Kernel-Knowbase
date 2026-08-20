---
title: "Shadow stacks for 64-bit Arm systems"
url: https://lwn.net/Articles/940403/
date: "August 7, 2023"
category: "Security-Control-flow integrity"
author: "By Jonathan Corbet August 7, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 7, 2023

[Return-oriented programming (ROP)](<https://en.wikipedia.org/wiki/Return-oriented_programming>) has, for some years now, been a valuable tool for those who would subvert a system's security. It is thus not surprising that a lot of effort has gone into thwarting ROP attacks, which depend on corrupting the call stack with a carefully chosen set of return addresses, at both the hardware and software levels. One result of this work is shadow stacks, which can detect corruption of the call stack, allowing the operating system to react accordingly. The 64-bit Arm implementation of shadow stacks is called "guarded control stack" (GCS); patches implementing support for this feature are currently under discussion. 

A shadow stack is a copy of a thread's call stack; it is often (but not necessarily) maintained by the CPU hardware. Whenever a function call is made, the current return address is pushed onto both the regular stack and the shadow stack. When the function returns, the addresses at the top of the two stacks are compared; if they do not match, the system concludes that the call stack has been corrupted and, probably, aborts execution. This check is enough to defeat most attacks that involve writing a sequence of return addresses to the stack. Even if the shadow stack is writable, the need to update it to match the call stack raises the bar for a successful exploit considerably. 

Software shadow stacks can be effective, but there are advantages to implementing them in hardware; the performance can be better, and the CPU can prevent attempts to corrupt the shadow stack. Naturally, any such support will be architecture-specific, and so will require architecture-specific code to make use of. The effort to implement [user-space shadow stacks for x86](<https://lwn.net/Articles/926649/>) has been underway for some time and will, with luck, land in the mainline in the near future. 

The 64-bit Arm ("aarch64") processors — and the developers adding support for the new processor features — are coming later to the shadow-stack party, a fact that brings both advantages and disadvantages. On the "advantage" side, the x86 developers have spent years in extended discussions over how shadow stacks should be supported and what the interface to them should look like, and they have the scars to show for it. In the cover letter for the [GCS support patch series](<https://lwn.net/ml/linux-kernel/20230716-arm64-gcs-v1-0-bf567f93bba6@kernel.org/>), Mark Brown made it clear that he intends to avoid a similar experience if possible: 

> As there has been extensive discussion with the wider community around the ABI for shadow stacks I have as far as practical kept implementation decisions close to those for x86, anticipating that review would lead to similar conclusions in the absence of strong reasoning for divergence. 

On the other hand, the first implementer of a kernel feature is not normally expected to make that implementation sufficiently general for the needs of those that will follow. Experience has shown that premature abstraction, like premature optimization, tends not to lead to good results. So it is often the second or third comer who has to create a framework that all implementations can fit into. 

#### The Arm shadow-stack interface

In this case, there was relatively little work of this type to do. The Arm world doesn't use the term "shadow stack" much, preferring the GCS term, but x86 got there first, so "shadow stack" has become the generic way of referring to this feature. The x86 implementation adds some [`arch_prctl()`](<https://man7.org/linux/man-pages/man2/arch_prctl.2.html>) calls to control the feature, but aarch64 does not implement `arch_prctl()` at all. So, instead, the GCS patches create a new set of [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) calls meant to control the feature on all architectures. The main control operation is `PR_SET_SHADOW_STACK_STATUS`, which takes a number of flags. 

Whenever a new thread is created, it will not have a shadow stack. One can be added using `PR_SET_SHADOW_STACK_STATUS` with the `PR_SHADOW_STACK_ENABLE` flag; that will cause a shadow stack to be allocated, and the calling thread will start using it. Since this call initializes the shadow stack, the portion of the call stack that was populated prior to the function that turned on the shadow stack will not be represented there; as a result, returning from that function will not be possible. Since the expectation is that the shadow stack will be enabled in the dynamic loader before jumping to a program's entry point, this limitation should not normally be a problem. 

Invoking `PR_SET_SHADOW_STACK_STATUS` _without_ `PR_SHADOW_STACK_ENABLE` set will disable the shadow stack. On the Arm architecture, the setup of a shadow stack can only be done once for any given thread; if the shadow stack is subsequently disabled, it is gone forever. 

Shadow-stack memory is specially marked and can be protected from manipulation by the owning process. There is a pair of flags that controls whether user space can make changes to the shadow stack (outside of those that happen automatically at function-call and return time). The `PR_SHADOW_STACK_PUSH` flag allows user space to push entries onto the stack using a special instruction, while `PR_SHADOW_STACK_WRITE` enables ordinary writes. Either or both of these capabilities may be needed to, for example, support user-space threading; enabling them reduces the security provided by the shadow stack, but the core defense against stack-smashing attacks (including ROP attacks) remains. 

As with x86, shadow stacks are normally allocated automatically by the kernel. In cases where user space may need to allocate shadow stacks separately (again, user-space threading comes to mind), the `map_shadow_stack()` system call is supported: 

```
void *map_shadow_stack(unsigned long addr, unsigned long size, unsigned int flags);
```

The returned pointer, on success, indicates a range of memory that has been properly prepared for shadow-stack use, with the protections set appropriately and the necessary tokens (which allow the CPU to recognize the stack and prevent concurrent use) put in place. Actually switching to the allocated stack requires using a dedicated Arm instruction. 

The `PR_LOCK_SHADOW_STACK_STATUS` flag locks the indicated configuration in place, preventing future changes. This flag can be used to prevent the thread from disabling the shadow stack or enabling writes to it. There is also a separate `PR_GET_SHADOW_STACK_STATUS` operation that can be used to query the current status. 

This patch set only implements shadow-stack support for user space; there is no support for kernel-space shadow stacks. 

#### Prospects

This work appears to be relatively uncontroversial and to be nearly ready to go, with one caveat: it depends on the x86 shadow-stack work in a number of ways. The x86 patches also seem nearly ready, but they were [turned down](<https://lwn.net/ml/linux-kernel/CAHk-=wiuVXTfgapmjYQvrEDzn3naF2oYnHuky+feEJSj_G_yFQ@mail.gmail.com/>) by Linus Torvalds during the 6.4 merge window, and were not proposed for 6.5. Until the x86 work lands in the mainline, the Arm patches will not be able do to so. As a result, the 6.7 release seems like the earliest that can be expected to include Arm shadow-stack support. 

It is also worth mentioning that shadow stacks are also coming to RISC-V, with the feature bearing the intuitive name "zisslpcfi". The [support patches](<https://lwn.net/ml/linux-kernel/20230213045351.3945824-1-debug@rivosinc.com/>) are still ""RFC quality"" and will likely need some work. They contain the generic `prctl()` operations (indeed, the first version of that interface appeared there), but do not include `map_shadow_stack()`, preferring an [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) interface that has been deemed unsuitable elsewhere. The zisslpcfi patches also include support for forward-edge control-flow integrity, similar to the x86 [indirect branch tracking](<https://lwn.net/Articles/889475/>). 

Hardware-based protection for control-flow integrity is clearly seen by all of the vendors as an important part of their security strategy, with most processors of interest adding support. Updating the kernel to actually use these features has been a slow process, with a number of roadblocks appearing along the way. The indications are, though, that this multi-year journey is reaching its end and attackers will have to move on to new techniques in the ongoing security arms race.
