---
title: Shadow stacks for user space
url: https://lwn.net/Articles/885220/
date: "February 21, 2022"
category: "Security-Control-flow integrity"
author: "By Jonathan Corbet February 21, 2022"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 21, 2022

The call stack is a favorite target for attackers attempting to compromise a running process; if an attacker finds a way to overwrite a return address on the stack, they can redirect control to code of their choosing, leading to a situation best described as "game over". As a result, a great deal of effort has gone into protecting the stack. One technique that offers promise is a shadow stack; support for shadow stacks is thus duly showing up in various processors. Support for protecting user-space applications with shadow stacks is taking a bit longer; it is currently under discussion within the kernel community, but adding this feature is trickier than one might think. Among other things, these patches have been around for long enough that they have developed some backward-compatibility problems of their own. 

#### Shadow-stack basics

Whenever one function calls another, information for the called function, including any parameters and the address to which the function should return once it has done its work, is pushed onto the call stack. As the call chain deepens, the chain of return addresses on the stack grows apace. Normally, all works as intended, but any corruption of the stack can cause one or more return addresses to be overwritten; that, in turn, will cause execution to "return" to an unintended location. With luck, that will cause the application to crash; if the corrupt data was deliberately placed there, instead, execution could continue in a way that will cause worse things to happen. 

Shadow stacks seek to mitigate this problem by creating a second copy of the stack that (usually) only contains the return-address data. Whenever a function is called, the return address is pushed onto both the regular stack and the shadow stack. When that function returns, the return addresses are popped off both stacks and compared; if they fail to match, the system goes into red alert and (probably) kills the process involved. Shadow stacks can be implemented entirely in software; even if the shadow stack is writable, it raises the bar for an attacker, who must now be able to corrupt two areas of memory, one of which is at an arbitrary location. Hardware support can make shadow stacks stronger, though. 

Intel processors, among others, can provide that support. If a shadow stack has been set up (which is a privileged operation), the pushing of return addresses onto that stack and comparison on function return are all done by the CPU itself. Meanwhile, the shadow stack is normally not writable by the application (other than by way of the function-call and return instructions), and thus not corruptible by an attacker. The hardware also requires the presence of a special "restore token" on the shadow stack itself that, among other things, ensures that two processes cannot be sharing the same shadow stack — a situation that, once again, would facilitate attacks. 

#### Supporting user-space shadow stacks

The current version of the shadow-stack support patches has been [posted by Rick Edgecombe](<https://lwn.net/ml/linux-kernel/20220130211838.8382-1-rick.p.edgecombe@intel.com/>); the bulk of the patches themselves were written by Yu-cheng Yu, who has posted numerous earlier versions of this work. Enabling this feature requires 35 non-trivial patches, and the problem is not entirely solved yet. One might wonder why it is so hard, since shadow stacks seem like a feature that most code could ignore almost all of the time, but life is never so simple. 

As might be expected, the kernel must contain the code to manage user-space shadow stacks. That includes enabling the feature at the processor level, and handing it for each specific process. Each process needs its own shadow stack set up with a proper restore token, then the (privileged) shadow-stack pointer register must be aimed at it. Faults, both normal page faults and things like integrity-violation traps, must be handled. There is yet more information to be managed on context switches. This is all pretty normal stuff for a new feature of this sort. 

The memory allocated for the shadow stack itself must be treated specially. It belongs to user space, but user-space code must not normally be allowed to write to it. The processor must also recognize memory dedicated to shadow stacks, so they must be marked specially in the page tables, and that's where things get a little interesting. There are a number of bits set aside in each page-table entry (PTE) to describe the protections that apply and various other types of status, but the x86 architecture does not include a "this is a shadow-stack page" bit. There _are_ some PTE bits set aside for the operating system's use; Linux does not use them all and could have spared one for this purpose, but evidently certain other operating systems have no spare PTE bits, so stealing one for this purpose would not be welcome. 

The solution that the hardware engineers arrived at might well be described as a bit of a hack. If a page's write-enable bit is clear (indicating that it cannot be written to), but its dirty bit is set (indicating that it _has_ been written to), the CPU will conclude that the page in question is part of a shadow stack. This is a combination of settings that, in ordinary usage, might not make sense, so it evidently seemed like fair game. 

Unfortunately, Linux kernel developers came to a similar conclusion many years ago, so Linux has its own interpretation for that combination of PTE bits. Specifically, that is how the kernel marks copy-on-write pages. The lack of write access will cause a trap should a process attempt to write the page; the presence of the dirty bit then tells the kernel to make a copy of the page and give the process write access to it. It all works well — until the CPU comes along and applies its own interpretation to that bit combination. So much of the patch set is focused on grabbing one of those unused PTE bits for a new `_PAGE_COW` flag and causing the memory-management code to use it. 

Shadow stacks bring other complications as well, of course. If a process calls [`clone()`](<https://man7.org/linux/man-pages/man2/clone.2.html>), a new shadow stack must be allocated for the child process; the kernel handles this task automatically. Signals, as always, add complications of their own, since they already involve various stack manipulations. It gets worse if a process has set up an alternative stack for signal handlers with [`sigaltstack()`](<https://man7.org/linux/man-pages/man2/sigaltstack.2.html>) — to the point that the current patch set does not handle that case at all. From such details (and more), a long patch series is made. 

#### ABI issues

The use of shadow stacks should be entirely transparent to most applications; after all, developers rarely think about the call stack in any case. But there will always be applications that do tricky things with their stacks, starting with multi-threaded programs that explicitly manage the stack area for each thread. Others may place their own specially crafted [thunks](<https://en.wikipedia.org/wiki/Thunk>) onto the stack or even more obscure things. Without special care, all of those applications will break if they are suddenly set up with a shadow stack. That sort of mass regression tends to make security features unpopular, so various measures have been taken to avoid it. 

The carefully considered plan that emerged was to mark applications (with a special property in the `.note.gnu.property` ELF section) that are prepared to run with a shadow stack. Applications that do no stack trickery could simply be rebuilt and run with shadow stacks thereafter. For the more complicated cases, a set of [`arch_prctl()`](<https://man7.org/linux/man-pages/man2/arch_prctl.2.html>) operations was defined to enable the explicit manipulation of shadow stacks. The GNU C Library was enhanced to use these calls to configure the environment properly on application startup, and the kernel would enable shadow stacks whenever a suitably marked program was run. Some distributions, including Fedora and Ubuntu, have been building their binaries for shadow stacks; all they need is a suitably equipped kernel to run with the extra protection. 

It is always dangerous to ship code using kernel features that have not yet been accepted and merged; shadow stacks turn out to be an example of why. According to the cover letter on the current series, the `arch_prctl()` API was ""abandoned for being strange"". But those shadow-stack-ready binaries deployed on systems worldwide were built expecting that API, strange or not, to be present; if the kernel respects the markings in the ELF file and enables shadow stacks for those programs, some of them will break. That would cause system administrators worldwide to disable shadow stacks until at least 2040, rather defeating the purpose of the whole exercise. 

One obvious workaround for this problem would be to never recognize the current ELF marker for shadow stacks and, instead, create a new one to mark binaries using the interface actually supported by the kernel. The decision that was made, though, was to get the kernel out of the business of recognizing shadow-stack-capable binaries entirely and let the C library take care of it. So, if this version of the ABI is adopted, the kernel will never enable shadow stacks unless user space requests it. 

#### The proposed interface

Overall control of shadow stack functionality is to be had with a (presumably not strange) `arch_prctl()` call: 

```
status = arch_prctl(ARCH_X86_FEATURE_ENABLE, ARCH_X86_FEATURE_SHSTK);
```

There is also an `ARCH_X86_FEATURE_DISABLE` operation that can be used to turn shadow stacks off, and `ARCH_X86_FEATURE_LOCK` to prevent future changes. 

While most applications need not worry about shadow stacks, some of them will need to be able to create new ones. Applications using [`makecontext()`](<https://man7.org/linux/man-pages/man3/makecontext.3.html>) and friends are a prominent example. Creating a shadow stack requires kernel support; the associated memory must have the special page bits set as described above, and must also include the restore token. So there is a new system call for this operation: 

```
void *map_shadow_stack(unsigned long size, unsigned int flags);
```

The size of the desired stack is passed as `size`, while `flags` has a single possibility: `SHADOW_STACK_SET_TOKEN` to request that a restore token be stored in the stack. The return value on success is the address of the base of this stack. 

Actually using this new stack is a matter of executing the `RSTORSSP` instruction to make the switch, most likely done as a part of a user-space context switch between threads. That instruction will perform the necessary verification of page permissions and the restore token before making the switch. It will also mark the token on the new shadow stack as being busy, preventing that stack from being used by any other process. 

Applications doing especially tricky things may require the ability to write to the shadow stack. That access is normally not allowed for obvious reasons but, as Edgecombe [noted](<https://lwn.net/ml/linux-kernel/20220130211838.8382-35-rick.p.edgecombe@intel.com/>), that ""restricts any potential apps that may want to do exotic things at the expense of a little security"". For the exotic case, another feature (`LINUX_X86_FEATURE_WRSS`) can be turned on with `arch_prctl()`; that, in turn, enables the `WRSS` instruction, which can write to shadow-stack memory. Directly writing to that memory by dereferencing a pointer is still disallowed in this case. 

#### What next?

This work is not exactly new; an early version of it was covered in [this 2018 article](<https://lwn.net/Articles/758245/>). In its previous incarnation, the shadow-stack patch set got up to [version 30](<https://lwn.net/ml/linux-kernel/20210830181528.1569-1-yu-cheng.yu@intel.com/>). The other half of the control-flow integrity work (indirect branch tracking), which got up to [version 29](<https://lwn.net/ml/linux-kernel/20210820182245.1188-1-yu-cheng.yu@intel.com/>) itself, has been set aside for the moment (though Peter Zijlstra has just [shown up](<https://lwn.net/ml/linux-kernel/20220218164902.008644515@infradead.org/>) with a separate implementation). With a new developer heading up the work, a reduction in scope, and some asked-for changes, it is hoped, this work can finally make some progress toward the mainline. 

In a number of ways, it looks like that hope might be realized. While there were comments on various parts of the patch set, there does not appear to be a lot of opposition to how it works at this point. Developers did express concern, though, about the lack of support for alternate signal stacks. That is a feature that seems certain to be wanted at some point, so it would be good to see how it fits into the whole picture before this functionality is merged. 

There was also [a separate subthread](<https://lwn.net/ml/linux-kernel/YgDIIpCm3UITk896@lisas.de/>) regarding problems with [Checkpoint/restore in user space](<https://criu.org/Main_Page>) (CRIU), which engages in no end of underhanded tricks to get its job done. One part of the checkpoint process involves injecting "parasite" code into a process to be checkpointed, grabbing the needed information, then doing a special return out of the parasite to resume normal execution. That is just the sort of control-flow tampering that shadow stacks are meant to prevent. Various possible solutions were discussed, but nothing has appeared in code form at this point. A solution here, too, seems necessary before shadow stacks can be merged; as Thomas Gleixner [put it](<https://lwn.net/ml/linux-kernel/87fsozek0j.ffs@tglx/>): ""We can't break CRIU systems with a kernel upgrade"". 

Finally, the range of supported hardware will almost certainly need to be expanded. Some AMD CPUs implement shadow stacks, evidently in a compatible manner, but only Intel CPUs are supported in this patch set; a lack of testing is cited as the reason. That, at least, will probably need to change for the work to go forward. Shadow stacks are also unsupported on 32-bit systems; fixing that may be harder and it is not clear whether the motivation to do that work exists. With or without 32-bit support, though, there is clearly still work to be done before this code enters the mainline. It should not be expected to show up in a near-future release.
