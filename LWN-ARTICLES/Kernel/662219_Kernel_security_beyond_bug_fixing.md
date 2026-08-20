---
title: "Kernel security: beyond bug fixing"
url: https://lwn.net/Articles/662219/
date: "October 28, 2015"
category: Security
author: "By Jonathan Corbet October 28, 2015 2015 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 28, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

As kernel security maintainer James Morris noted in the introduction to a 2015 Kernel Summit session, a lot of progress has been made with regard to kernel security in the last 10-15 years. That said, there are lot of things we could be doing better, and one could make the case that we have fallen behind the state of the art in a number of areas, including self-protection and hardening. On that note, he stepped aside and let Kees Cook give the group the bad news about what needs to be done to improve the kernel's security. 

Kees started by making the claim that security needs to be more than just access control and attack-surface reduction. Crucially, it also needs to be more than just fixing bugs. The kernel needs to learn to protect itself better in the presence of inevitable security bugs, even if that means imposing some pain on kernel developers. 

There are, Kees said, one billion Android devices in circulation. Most of them are running 3.4 kernels, with the (still old) 3.10 kernel running a distant second. That, he said, is "completely terrifying." The lifetime of critical security bugs is huge; bugs are often found many years after [![\[Kees Cook\]](https://static.lwn.net/images/conf/2015/klf-ks/KeesCook-sm.jpg)](<https://lwn.net/Articles/662227/>) they have been introduced into the kernel. But attackers are often finding these bugs right away and exploiting them for most of those years while they remain in the kernel. 

We _are_ finding bugs, especially with the introduction of more checkers and such. We are fixing the bugs. But there will always be bugs because we keep writing them. It's a whack-a-mole situation, but playing whack-a-mole is not the solution. Instead, we need to get to a point where we can handle failures safely. It is not overstating things to say that, in an era of things like self-driving cars, lives depend on our solving this problem. 

The best way to deal with security problems is to kill entire classes of exploits at a time. Getting there involves eliminating targets, methods of attack, information leaks, or anything else that helps attackers. We have to do that, even if it makes life more difficult for developers. This stuff _will_ get in people's way; that makes it a hard sell. 

To eliminate classes of attack, we need to understand how typical exploit chains work. Most attacks exploit more than one flaw in the target system. At various times they need to know where the targets are, inject malicious code into the system, find where that code ended up, and redirect control to that code. Each of those may require exploiting a different flaw. Attackers often have a number of flaws that can be exploited to carry out any given step in the chain; if one flaw is fixed, another can be used instead. 

#### Dealing with vulnerabilities

So what can we do? Kees launched into a series of vulnerabilities and steps that might be taken to find and eliminate entire classes of them. 

First on the list was stack overflows. A classic approach for the detection of stack overflows is putting a canary on the stack, but there are some exploits that write far beyond the end of the stack, skipping over the canary entirely. Stack location randomization can help here, as can shadow stacks — parallel stacks where important values like return addresses are stored. 

Integer overflows and underflows are the source of many vulnerabilities. For these, it is possible to instrument the compiler to detect overflows at run time. This process is not free of pain; sometimes overflows are expected, so the compiler must be told that the code is correct. Interestingly, this instrumentation can, at times, actually improve the performance of the code. 

Heap overflows can be addressed by runtime validation of variable sizes in the `copy_*_user()` functions and elsewhere. Placement of guard pages can catch heap-overflow exploits. Runtime validation of linked lists is also a useful technique here. 

For format-string injection problems, the best thing to do would be to drop the `%n` format specifier entirely. (That specifier causes the number of characters written to be stored in a variable; it's worth noting that the kernel's format-string handling already ignores `%n`). 

Kernel pointer leaks are everywhere; the [`kptr_restrict`](<https://lwn.net/Articles/420403/>) mechanism is far too weak. It requires developers to explicitly opt in to prevent pointer leaks, so many don't. A more useful technique would be, for example, to instrument the [seq_file subsystem](<https://lwn.net/Articles/22355/>) to detect use of `%p` (used to format pointers) and simply block output when somebody tries to use it. 

Uninitialized variables can be mitigated by clearing the kernel stack between system calls. As Kees described in [a talk [PDF]](<https://outflux.net/slides/2011/defcon/kernel-exploitation.pdf>) some years ago, uninitialized variables on the stack can be exploitable. 

#### Blocking exploits

Many exploits require finding the location of the kernel in physical memory, so anything that can be done to make the kernel harder to find will make those exploits harder. This can be done by hiding symbols and avoiding the leaking of kernel pointers to user space. [Kernel address-space layout randomization](<https://lwn.net/Articles/569635/>) is not a perfect shield, but it can still help to make finding the kernel harder. Setting memory protections so that executable pages cannot be read (if the hardware supports this) can be a good technique. Kees also suggested build-time structure layout randomization. 

Exploits can overwrite kernel text directly — something that, Kees said, should not be possible at all. Ensuring that executable pages are not writable would help. There are techniques in the kernel (jump labels, for example) that depend on being able to overwrite code; they can still be used by mapping in new pages or simply turning on write permissions for as long as it takes to make the change. Moving away from situations where a single write instruction can compromise the kernel will make us more secure. 

Overwriting of function pointers can be blocked by making tables (and structures) of function pointers `const`. This has been done in parts of the kernel, but there are many more opportunities for improvement there. 

The ability to make the kernel execute code in user-space memory is exploitable. The best solution here can be hardware segmentation; Intel's "supervisor-mode execution prevention" and ARM's "privileged execute never" can both block execution from user-space memory. Instrumenting the compiler to set the high address bit on all kernel function calls can block calls into user-space memory (since the kernel's address space is at the upper end of the virtual address range, while user space is at the bottom). Kees also suggested emulating segmentation by using separate page tables for user mode and kernel mode; Linus jumped in at this point to say that this is the kind of idea that makes security people look crazy; such an approach would never perform well. He suggested avoiding talking about ideas that will clearly never make it into the mainline. 

[Return-oriented programming](<https://en.wikipedia.org/wiki/Return-oriented_programming>) can be used to piece together desired functionality out of chunks of existing code. This kind of code-chunk reuse can be fought with compiler instrumentation to ensure "control-flow integrity." 

#### Challenges

Even if we know how to deal with many classes of exploits, there are non-technical challenges that get in the way. At the top of this list is conservatism. It took 16 years, for example, to get basic [symbolic link protections](<https://lwn.net/Articles/390323/>) into the kernel, and that was just providing a defense for user space. We as a community have to accept that we need these features, even though some of them are going to be a burden. 

Another challenge is the additional complexity that comes with many security technologies. But, Kees said, we have done many complex things over the years; we can handle this one as well. 

Finally, there is the challenge of resources. To get this work done we need developers, testers, backporters, and more. These need to be people who are dedicated to those roles, meaning that it needs to be paid work. This is an industry-wide problem; companies working in this industry need to support work on the solutions. 

The kernel community has often been hostile to changes that increase security if they decrease usability or performance, or if they make development harder. But this particular talk led to a lot of discussion among the attendees. It would seem that the kernel development community is coming around to the idea that some sacrifices may need to be made to provide the level of security that our users need. The real test will come when the patches start to arrive; if, as Kees suggested, developers manage to avoid reflexively rejecting security patches, things will have started moving in the right direction. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Kernel Summit].
