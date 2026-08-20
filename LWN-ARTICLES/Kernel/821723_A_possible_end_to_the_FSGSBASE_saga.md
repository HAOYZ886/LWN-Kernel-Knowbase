---
title: A possible end to the FSGSBASE saga
url: https://lwn.net/Articles/821723/
date: "June 1, 2020"
category: "Architectures-x86"
author: "By Jonathan Corbet June 1, 2020"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 1, 2020

The [FSGSBASE patch series](<https://lwn.net/ml/linux-kernel/20200528201402.1708239-1-sashal@kernel.org/>) is up to its thirteenth version as of late May. It enables some "new" instructions for the x86 architecture, opening the way for a number of significant performance improvements. One might think that such a patch series would be a shoo-in, but FSGSBASE has had a troubled history; meanwhile, the delays in getting it merged may have led to a number of users installing root holes on their Linux systems in the hope of improving security. 

"Segments" are a holdover from ancient versions of the x86 architecture; they once were distinct regions of memory used to get around the addressing limitations of that era. Virtual memory has done away with the need for segments, but the concept persists; x86_64 processors only implement two of the original segments (called "FS" and "GS"). In these processors, a "segment" is really just an offset into virtual memory with little other meaning; their remaining value comes from the segment-based addressing mode supported by the CPU. 

Historic or not, these segment registers are still used. A common use for FS in user space is thread-local storage; each thread has a unique value of the FS base register pointing to its own storage area. Code running in threads can then use segment-based addressing to access local storage without having to worry about where that storage is. The kernel, instead, uses GS in a similar way for per-CPU data. There are some relics of the kernel's one-time use of FS to indicate the address range accessible to user space, but the kernel's [`get_fs()` and `set_fs()` functions](<https://lwn.net/Articles/722267/>) no longer use that segment. 

Modifying the segment registers has always been a privileged operation. There is value, though, in letting user space make use of the FS and GS base registers, so the kernel provides that functionality via the [`arch_prctl()` system call](<https://www.man7.org/linux/man-pages/man2/arch_prctl.2.html>). Since the base registers are actually set by the kernel, privileged code can count on knowing what their contents will be (and that said contents make sense). 

#### FSGSBASE

Calling into the kernel to set a register is a relatively expensive operation, though. If the call needs to be done once to set up a thread-local storage area, nobody will notice the cost, but code needing to make frequent changes to the FS or GS base register will be slowed down by the system-call overhead. Actually setting those registers, which are stored in x86 model-specific registers (MSRs), is somewhat costly in its own right. So Intel added a set of instructions to directly manipulate the FS and GS base registers to the "Ivy Bridge" series of processors in 2012. This set of instructions is often referred to as "FSGSBASE". 

Before user space can actually use those instructions, though, the kernel must set a special bit enabling them and, despite the time that has passed since they became available, that bit remains unset. Since the kernel has always had control of those registers, it contains a number of assumptions about their contents; just letting user space change them without preparing the kernel first is a recipe for any of a number of easily exploited vulnerabilities. 

Avoiding those problems is conceptually fairly simple, though a bit more complex in the implementation. The kernel must take pains to ensure that the FS and (especially) GS registers have correct values on every entry into kernel space. The handling of [certain speculative-execution vulnerabilities](<https://lwn.net/Articles/795637/>) gets a bit more complicated. And, of course, a control knob must be provided so that administrators can turn FSGSBASE off if need be. 

All it takes is somebody to write this code. Intel was slow to post FSGSBASE patches — and nobody else stepped forward to do that work either. When patches were finally posted, they ran into a number of problems in review and have required numerous revisions. The curious can see [this message from Thomas Gleixner](<https://lore.kernel.org/lkml/alpine.DEB.2.21.1903261010380.1789@nanos.tec.linutronix.de/>) for an opinionated timeline of events through March 2019. [Version 7 of the patch set](<https://lwn.net/ml/linux-kernel/1557309753-24073-1-git-send-email-chang.seok.bae@intel.com/>), posted in May 2019, got as far as being merged into the x86 subsystem tree before various problems came to light; that merge was [subsequently reverted](<https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git/commit/?h=x86/cpu&id=049331f277fef1c3f2527c2c9afa1d285e9a1247>) in a rather grumpy way after new problems came to light. More recently, Sasha Levin has picked up this work (despite not being an Intel employee) and is trying to get it across the line; he may yet succeed for the 5.9 development cycle. 

#### Root holes enter a vacuum

The development of this seemingly simple feature has been a rather long and fraught process; during all of this time, users have been unable to take advantage of it. But users, being users, have proved unwilling to wait. One of the use cases that has created the most pressure is Intel's "[Software Guard Extensions](<https://software.intel.com/content/www/us/en/develop/topics/software-guard-extensions.html>)" (SGX), which is meant to allow the creation of private "enclaves" to protect privileged code and data. SGX support for the kernel has had [its own difficult history](<https://lwn.net/Articles/786487/>) and remains unmerged, so developers wanting to figure out how to make use of this feature have been working entirely out of tree. 

One of the most prominent projects in this area is [Graphene](<https://grapheneproject.io/>), which describes itself as a "library OS" for secure applications. The web site mentions SGX this way: 

Graphene runs unmodified applications inside Intel SGX. It supports dynamically loaded libraries, runtime linking, multi-process abstractions, and file authentication. For additional security, Graphene performs cryptographic and semantic checks at untrusted host interface. Developers provide a manifest file to configure the application environment and isolation policies, Graphene automatically does the rest. 

Graphene got its start as a research project, but has since received a fair amount of support from companies, including Intel. The project has ambitions for being the standard SGX support platform, and some cloud providers are evidently [looking at supporting it](<https://www.alibabacloud.com/blog/protecting-go-language-applications-with-the-graphene-library-os-on-intel%C2%AE-sgx%C2%AE-secured-alibaba-cloud_594889>) — with Intel's blessing. 

There is just one little problem with Graphene: working with SGX requires the ability to modify the FS base register frequently. To keep calls into enclaves fast, Graphene loads [a little kernel module](<https://github.com/oscarlab/graphene-sgx-driver/blob/master/gsgx.c>) that enables the FSGSBASE instructions. Since the kernel is not prepared for this, that action immediately opens up a root hole on the system involved — just what one wants to see from a system that is supposed to bring heightened security. Graphene is not alone in this behavior; the [Occlum SGX library](<https://github.com/occlum/occlum>) does the same thing, for example. 

It is fair to say that the kernel-development community was, as a whole, unimpressed by this approach to the problem. Don Porter, one of the creators of Graphene, [tried to justify](<https://lwn.net/ml/linux-kernel/c5fffcd1-c262-7046-a047-67de2bbccd78@cs.unc.edu/>) enabling FSGSBASE behind the kernel's back by pointing out that SGX projects assume that the host operating system is compromised; SGX exists to protect data in just that situation, after all. Extending that philosophy to compromising the system from the outset, though, is still a hard sell. 

In the end, kernel developers can usually understand the idea of using this kind of a hack to get a problem out of the way while addressing other issues. The fact that there is no warning to be found in Graphene's flashy web site, or in the papers describing Graphene, that installing the code compromises the system is harder to swallow. There is even, as Levin [pointed out](<https://lwn.net/ml/linux-kernel/20200518151604.GC33628@sasha-vm/>), a book called _Responsible Genomic Data Sharing_ that suggests using Graphene, which seems not entirely responsible. After some discussion, Porter [came around to the idea](<https://lwn.net/ml/linux-kernel/39df26f2-9d17-b0f2-6a55-48195542637b@cs.unc.edu/>) that some high-profile warnings are needed to keep potential users from opening up their systems in the name of "security". 

Warnings are a step in the right direction, but the proper way to address this problem is to get the FSGSBASE patches into the kernel so that the other hacks are no longer necessary. As an added benefit, these patches make the kernel faster too, since the new instructions are faster than performing operations on MSRs. Proper FSGSBASE support should, thus, make almost everybody happier. 

As noted, that will hopefully happen soon. This whole long affair has left a bit of a bad taste in many developers' mouths, though; there is some overt unhappiness with how Intel has handled the situation. Straightening that out may take longer.
