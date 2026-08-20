---
title: "What's the best way to prevent kernel pointer leaks?"
url: https://lwn.net/Articles/735589/
date: "October 5, 2017"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet October 5, 2017"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 5, 2017

An attacker who seeks to compromise a running kernel by overwriting kernel data structures or forcing a jump to specific kernel code must, in either case, have some idea of where the target objects are in memory. Techniques like kernel address-space layout randomization have been created in the hope of denying that knowledge, but that effort is wasted if the kernel leaks information about where it has been placed in memory. Developers have been plugging pointer leaks for years but, as a recent discussion shows, there is still some disagreement over the best way to prevent attackers from learning about the kernel's address-space layout. 

There are a number of ways for a kernel pointer value to find its way out to user space, but the most common path by far is the `printk()` function. There are on the order of 50,000 `printk()` calls in the kernel, any of which might include the value of a kernel pointer. Other places in the kernel use the underlying `vsprintf()` mechanism to format data for virtual files; they, too, often leak pointer values. A blanket ban on printing pointer values could solve this problem — if it could be properly enforced — but it would also prevent printing such values when they are really needed. Debugging kernel problems is one obvious use case for printing pointers, but there are others. 

The approach that has been taken in the kernel is to try to identify the places where kernel pointers are printed and, perhaps, censor that information on its way to user space. The special "`%pK`" formatting directive ([added in 2011](<https://git.kernel.org/linus/455cd5ab305c>)) should be used to print kernel pointers; the formatting code will, among other things, be sure to format them correctly regardless of the architecture the kernel is running on. This directive also interacts with the `kptr_restrict` sysctl knob, though. If that knob is set to zero (as it is by default), kernel addresses are printed unchanged. Setting it to one will cause kernel pointers to be printed as all zeroes unless the current process is running with privilege; setting it to two wipes all kernel addresses unconditionally. 

One can immediately pick out some shortcomings in this scheme. It is an opt-in mechanism that depends on all kernel developers properly marking the places where they print kernel pointers. It depends on the credentials of the running process; that makes sense for situations like reading a `/proc` file (which doesn't use `printk()` but does use the underlying formatting support), but it's less useful for the many places in the kernel that call `printk()` in response to an asynchronous event. It also can allow other types of possibly sensitive addresses (physical addresses, for example) to be exposed. 

Tobin Harding recently tried to improve the situation with [a patch set](<https://lwn.net/Articles/735596/>) tightening up the printing of pointer values in general. It made a few specific changes: 

  * It adds two new values for `kptr_restrict`. A setting of three will prevent the printing of pointer values with the unadorned "`%p`" directive. In theory, no kernel pointers should be printed that way, but the real world is not so ideal. Setting `kptr_restrict` to four will also prevent the printing of physical address values (those printed with "`%pa`", "`%pr`", and "`%pR`"). 

  * The default value of `kptr_restrict` is changed to four as a way of preventing address leaks during the early boot process. 

  * A new "`%pP`" directive indicates a pointer value that should always be printed regardless of the setting of `kptr_restrict`. The initial use for this directive is in the printing of stack traces. 

  * There is also a new unconditional version of "`%pa`" ("`%paP`", along with "`%padP`" for DMA addresses and "`%papP`" as a synonym for "`%paP`"). Some user-space UIO drivers need that information. 

There were some immediate concerns about defaulting `kptr_restrict` to the most restrictive setting. It seems certain to make life difficult for developers trying to debug problems that show up early in the bootstrap process. As Linus Torvalds [noted](<https://lwn.net/Articles/735599/>), that could lead to developers circumventing the mechanism entirely by using something like "`%x`" to print pointer values. Options like setting the default value in the kernel configuration or on the command line were discussed, but the discussion quickly took a different turn. 

Torvalds also complained that the entire `kptr_restrict` mechanism is the wrong approach to the problem. The read-time capability test does not always make sense and, he [said](<https://lwn.net/Articles/735600/>), a global switch is the wrong way to handle the problem. Attempts to make that switch more restrictive by default [have run into trouble](<https://lwn.net/Articles/443379/>) in the past and have been backed out as a result. The proper solution, Torvalds said, is to simply fix all of the places in the kernel that leak addresses. 

That is, of course, easier said than done, as Jordan Glover [remarked](<https://lwn.net/Articles/735601/>): ""If we knew where those leaks are hiding they will be fixed already"". It is better, he said, to assume that there will be leaks in the kernel and try to block them all at once. But Torvalds believes that the same effect as a restrictive `kptr_restrict` setting could be achieved by searching for (and fixing) every use of unadorned "`%p`" directives in the kernel. It would be a fair amount of work, but perhaps much of it could be scripted. 

He also [suggested](<https://lwn.net/Articles/735602/>) that many of the problems could be found by searching for addresses showing up in actual log files. Companies like Google, he said, probably have a lot of kernel logs sitting around; searching them for addresses should quickly reveal the real problems, which can then be fixed. He demonstrated with the log from his own system, which included a physical address printed with "`%x`" and which, thus, would not have been redacted by the proposed patches. And there are even more paths for kernel addresses to leak; he [mentioned](<https://lwn.net/Articles/735603/>) a case where the netfilter code was using an address in a slab name, which then showed up in the kernel's slab statistics. 

Some patch submitters would have been dismayed by this response. Harding, instead, [responded](<https://lwn.net/Articles/735604/>) that this project ""sounds like just the job for an upcoming kernel hacker, with a lot of time and not much experience, to do something laborious that no one else wants to do and learn a bunch about the kernel."" It seems that he is that kernel hacker; he went on to propose dropping the patch set in favor of tracking down and fixing the actual leaks, and added ""I'm super keen to work"". 

So that seems to be the likely direction for this work. Some of the existing patches will probably get into the kernel eventually, though; there is value in identifying the types of all addresses being printed. Maybe, someday, the bare "`%p`" directive will disappear. That will not happen in the immediate future, though; there are a lot of call sites to fix first.
