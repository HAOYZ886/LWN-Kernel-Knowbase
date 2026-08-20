---
title: Intel AMX support in 5.16
url: https://lwn.net/Articles/874846/
date: "November 8, 2021"
category: "Architectures-x86; Releases-5.16"
author: "By Jonathan Corbet November 8, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 8, 2021

The x86 instruction set is large, but that doesn't mean it can't get bigger yet. Upcoming Intel processors will feature a new set of instructions under the name of "Advanced Matrix Extensions" (AMX) that can be used to operate on matrix data. After a somewhat bumpy development process, support for AMX has found its way into the upcoming 5.16 kernel. Using it will, naturally, require some changes by application developers. 

AMX (which is described in [this document](<https://software.intel.com/content/dam/develop/external/us/en/documents-tps/architecture-instruction-set-extensions-programming-reference.pdf>)) is meant to be a general architecture for the acceleration of matrix operations on x86 processors. In its initial form, it implements a set of up to eight "tiles", which are arrays of 16 64-byte rows. Programmers can store matrices in these tiles of any dimension that will fit therein; a matrix of 16x16 32-bit floating-point values would work, but other geometries are supported too. The one supported operation currently will multiply the matrices stored in two tiles, then add the result to a third tile. By chaining these operations, multiplication of matrices of any size can be implemented. Evidently other operations are meant to come in the future. 

While AMX may seem like a feature aimed at numerical analysis, the real target use case would appear to be machine-learning applications. That would explain why 16-bit floating-point arithmetic is supported, but 64-bit is not. 

The design of AMX gives the kernel control over whether these features can be used by any given process. There are a couple of reasons for this, one being that AMX instructions, as one might imagine, use a lot of processor resources. A process doing heavy AMX work on a shared computer may adversely affect other processes. But AMX also cannot be supported properly unless both the kernel and the user-space process are ready for it. 

#### Development process

Support for AMX was first [posted](<https://lwn.net/ml/linux-kernel/20201001203913.9125-1-chang.seok.bae@intel.com/>) by Chang Bae in October 2020, but got relatively few review comments. By the time [version 4](<https://lwn.net/ml/linux-kernel/20210221185637.19281-1-chang.seok.bae@intel.com/>) came out in February, more developers were paying attention, and they were not entirely pleased with how this feature was being integrated into the kernel's existing floating-point unit (FPU) code. Various versions followed, with the frustration level seeming to increase; at the end of September, Len Brown posted [minutes from a conversation](<https://lwn.net/ml/linux-kernel/CAJvTdKkK=_pp1PrWdh1_GN73VifuAkivnErgK+bo2h34Vd_55w@mail.gmail.com/>) that, seemingly, showed a path forward. 

Unfortunately, [version 11](<https://lwn.net/ml/linux-kernel/20211001223728.9309-1-chang.seok.bae@intel.com/>), posted the very next day, seemed to ignore many of the decisions that had been made. This posting drew [a sharp rebuke](<https://lwn.net/ml/linux-kernel/87k0iuhq8b.ffs@tglx/>) from Thomas Gleixner, who felt that the feature was being shoved into the kernel without listening to the complaints that were being raised. Things weren't looking great for AMX, but work was happening behind the scenes; in mid-October, Gleixner posted [a massive reworking of the FPU code](<https://lwn.net/ml/linux-kernel/20211011215813.558681373@linutronix.de/>) meant to ease the task of supporting AMX. [A new AMX patch set](<https://lwn.net/ml/linux-kernel/20211021225527.10184-1-chang.seok.bae%40intel.com/>) followed shortly thereafter, and that, more or less, is what ended up in 5.16. 

Gleixner's [pull request](<https://lwn.net/ml/linux-kernel/163572865296.3357115.3707320162730818106.tglx%40xen13/>) for this code acknowledged its relatively unseasoned nature: 

> Note, this is relatively new code despite the fact that AMX support is in the works for more than a year now. 
> 
> The big refactoring of the FPU code, which allowed to do a proper integration has been started exactly 3 weeks ago. Refactoring of the existing FPU code and of the original AMX patches took a week and has been subject to extensive review and testing. The only fallout which has not been caught in review and testing right away was restricted to AMX enabled systems, which is completely irrelevant for anyone outside Intel and their early access program. There might be dragons lurking as usual, but so far the fine grained refactoring has held up and eventual yet undetected fallout is bisectable and should be easily addressable before the 5.16 release. Famous last words... 

The FPU code is relatively tricky, low-level stuff, so it would indeed be unsurprising to find a dragon or two lurking in the new work. 

#### Using AMX

As noted above, the kernel is able to control which processes are able to use the AMX instructions. The first step for a user-space process would be to use a new [`arch_prctl()`](<https://man7.org/linux/man-pages/man2/arch_prctl.2.html>) command (`ARCH_GET_XCOMP_SUPP`) to get a list of supported features; if the appropriate bit is set in the result, AMX is available. Then, another `arch_prctl()` command (`ARCH_REQ_XCOMP_PERM`) can be used to request permission to use AMX. Some checks are made (one to be described shortly), and there is an opportunity for security modules to express an opinion as well. Normally, though, the request will be granted. Permissions apply to all threads in a process and are carried over a fork; calling `execve()` will reset them, though. 

One challenge presented by AMX is that processors can create a great deal of internal state while the AMX instructions are running. If the CPU is interrupted in the middle of an operation, that state must be saved somewhere or a lot of work could be lost. So, if a process is using AMX, the kernel must be prepared to save up to about 10KB of data in its interrupt handlers before doing much of anything else. This saving is done using the `XSAVE` instruction. 

The kernel allocates memory for each process that can be used for this purpose. Allocating 10KB for every process in the system would waste a lot of memory, though; most processes will never use AMX instructions. Happily, the processor can be configured to trap into the kernel the first time any process executes an AMX instruction; the kernel can then check whether permission to use those instructions has been granted and, if so, allocate an appropriately sized buffer to hold the FPU state and allow the operation to continue. 

One potential problem has to do with the [`sigaltstack()` system call](<https://man7.org/linux/man-pages/man2/sigaltstack.2.html>), which allows a thread to establish a new stack for signal handling. That stack, too, must be large enough to hold the FPU state if the process involved is using AMX. For years, developers have been told to use `MINSIGSTKSZ` as the minimum size for this stack; that size, which is 2KB, is nowhere near large enough for AMX-using processes. Indeed, it is not even large enough to use the [AVX-512 extensions](<https://en.wikipedia.org/wiki/AVX-512>), a fact that has caused some corrupted-stack problems in the past. 

To avoid this problem for AMX, the kernel will check to ensure that all signal stacks are sufficiently large. This check is done at each call to `sigaltstack()`, but a check of existing stacks is also done when a process requests permission to use AMX in the first place. Processes not using AMX will not need the larger stack and, thus, will not be broken by these checks. Processes that _do_ want to use AMX will not be allowed to unless all of their signal stacks are big enough. 

Once the infrastructure to perform these checks was put in place, the kernel also gained the ability to ensure that processes using AVX-512 have adequately sized signal stacks. Enforcing that condition, though, has the potential to break applications that seemingly work now, perhaps because their signal handlers are never actually called. To avoid this problem, there is a kernel configuration option (`STRICT_SIGALTSTACK_SIZE`) and a command-line option (`strict_sas_size=`), either of which can be used to control whether the strict checks are carried out when AVX-512 is in use. 

Assuming that all of the pieces hold together, this is the form that AMX support will take in 5.16. Those wanting more information can look at the commits containing [AMX test cases](<https://git.kernel.org/linus/6a3e0651b4a0>) and [some documentation](<https://git.kernel.org/linus/d7a9590f608d>) on the `arch_prctl()` commands. Meanwhile, keep an eye out for dragons for the next nine weeks or so.
