---
title: "Preventing stack guard-page hopping"
url: https://lwn.net/Articles/725832/
date: "June 19, 2017"
category: "Security-Vulnerabilities"
author: "By Jonathan Corbet June 19, 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 19, 2017

Normally, the -rc6 kernel testing release is not the place where one would expect to find a 900-line memory-management change. As it happens, though, such a change was quietly merged immediately prior to the [4.12-rc6](<https://lwn.net/Articles/725786/>) release; indeed, it may have been the real reason behind 4.12-rc6 coming out some hours later than would have been expected. This change is important, though, in that it addresses a newly publicized security threat that, it seems, is being actively exploited. 

**A correction** : Ben Hutchings pointed out that the Qualys analysis is based on the "main thread" stack, not any other thread stacks which, with glibc at least, are not allowed to grow. Apologies for the confusion. 

The stack area in a running process is, on most architectures, placed at a relatively high virtual address; it grows downward as the process's stack needs increase. A virtual-memory region that automatically grows as a result of page faults brings some inherent risks; in particular, it must be prevented from growing into another memory region placed below it. In a single-threaded process, the address space reserved for the stack can be large and difficult to overflow. Multi-threaded processes contain multiple stacks, though; those stacks are smaller and are likely to be placed between other virtual-memory areas of interest. An accidental overflow could corrupt the area located below a stack; a deliberate overflow, if it can be arranged, could be used to compromise the system. 

The kernel has long placed a guard page — a page that is inaccessible to the owning process — below each stack area. (Actually, it hasn't been all that long; the guard page [was added in 2010](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=320b2b8de12698082609ebbc1a17165727f4c893>)). A process that wanders off the bottom of a stack into the guard page will be rewarded with a segmentation-fault signal, which is likely to bring about the process's untimely end. The world has generally assumed that the guard page is sufficient to protect against stack overflows but, it seems, the world was mistaken. 

On June 19, Qualys [disclosed](<https://blog.qualys.com/securitylabs/2017/06/19/the-stack-clash>) a set of vulnerabilities that make it clear that a single guard page is not sufficient to protect against stack overflow attacks. These vulnerabilities have been dubbed "Stack Clash"; the associated domain name, logo, and line of designer underwear would appear to not have been put in place yet. This problem has clearly been discussed in private channels for a while, since a number of distributors were immediately ready with kernel updates to mitigate the issue. 

The fundamental problem with the guard page is that it is too small. There are a number of ways in which the stack can be expanded by more than one page at a time. These include places in the GNU C Library that make large [`alloca()`](<http://man7.org/linux/man-pages/man3/alloca.3.html>) calls and programs with large variable-length arrays or other large on-stack data structures. It turns out to be relatively easy for an attacker to cause a program to generate stack addresses that hop over the guard page, stomping on whatever memory is placed below the stack. The proof-of-concept attacks posted by Qualys are all local code-execution exploits, but it seems foolhardy to assume that there is no vector by which the problem could be exploited remotely. 

[The fix merged for 4.12](<https://git.kernel.org/linus/1be7107fbe18eed3e319a6c3e83c78254b693acb>) came from Hugh Dickins, with credit to Oleg Nesterov and Michal Hocko. It takes a simple, arguably brute-force approach to the problem: the 4KB guard page is turned into a 1MB guard region on any automatically growing virtual memory area. As the patch changelog notes: ""It is obviously not a full fix because the problem is somehow inherent, but it should reduce attack space a lot."" The size of the guard area is not configurable at run time (that can wait until somebody demonstrates a need for it), but it can be changed at boot time with the `stack_guard_gap` command-line parameter. 

The 1MB guard region should indeed be difficult to jump over. It is (or should be) a rare program that attempts to allocate that much memory on the stack, and other limits (such as the limit on command-line length) should make it difficult to trick a program into making such an allocation. On most 64-bit systems, it should be possible to make the guard region quite a bit larger if the administrator worries that 1MB is not enough. Doubtless there are attackers who are feverishly working on ways to hop over those regions but, for a while at least, they may well conclude that there are easier ways to attack any given system. 

The real problem, of course, is that a stack pointer can be abused to access memory that is not the stack. Someday, perhaps, we'll all have memory-type bits in pointers that will enable the hardware to detect and block such attacks. For now, though, we all need to be updating our systems to raise the bar for a successful compromise. Distributors have updates now, and the fix is in the queue for the next round of stable kernel updates due on June 21.
