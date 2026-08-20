---
title: "Blocking userfaultfd() kernel-fault handling"
url: https://lwn.net/Articles/819834/
date: "May 8, 2020"
category: "Security-Kernel hardening; userfaultfd"
author: "By Jonathan Corbet May 8, 2020"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 8, 2020

The [`userfaultfd()`](<http://www.man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call is a bit of a strange beast; it allows user space to take responsibility for the handling of page faults, which is normally a quintessential kernel task. It is thus perhaps not surprising that it has turned out to have some utility for those who would attack the kernel's security as well. A recent [patch set](<https://lwn.net/ml/linux-kernel/20200423002632.224776-1-dancol@google.com/>) from Daniel Colascione is small, but it makes a significant change that can help block at least one sort of attack using `userfaultfd()`. 

A call to `userfaultfd()` returns a file descriptor that can be used for control over memory management. By making a set of `ioctl()` calls, a user-space process can take responsibility for handling page faults in specific ranges of its address space. Thereafter, a page fault within that range will generate an event that can be read from the file descriptor; the process can read the event and take whatever action is necessary to resolve the fault. It should then write a response describing that resolution to the same file descriptor, after which the faulting code will resume execution. 

This facility is normally intended to be used within a multi-threaded process, where one thread takes on the fault-handling task. There are a number of use cases for `userfaultfd()`; one of the original cases was handling live migration of a process from one machine to another. The process can be moved and restarted on the new system while leaving most of its memory behind; the pages it needs immediately can then be demand-faulted across the net, driven by `userfaultfd()` events. The result is less downtime while the process is being moved. 

Since the kernel waits for a response from the user-space handler to resolve a fault, page faults can cause an indefinite delay in the execution of the affected process. That is always the case, of course; for example, a process generating a fault on memory backed by a file somewhere else on the network will come to an immediate halt for an unknown period of time. There is a difference with `userfaultfd()`, though: the time it takes to resolve the fault is under the process's direct control. 

Normally, there are no problems that can result from that control; the process is simply slowing itself down, after all. But occasionally page faults will be generated in the kernel. Imagine, for example, just about any system call that results in the kernel accessing user-space memory. That can happen as the result of I/O, from a `copy_from_user()` call, or any of a number of other ways. Whenever the kernel accesses user-space memory, it has to be prepared for the relevant page(s) to not be present; the kernel has to incur and handle a page fault, in other words. 

An attacker can take advantage of this behavior to cause execution in the kernel to block at a known point for a period of time that is under said attacker's control. In particular, the attacker can use `userfaultfd()` to take control of a specific range of memory; they then ensure that none of the pages in that range are resident in RAM. When the attacker makes a system call that tries to access memory in that range, they will get a `userfaultfd()` event helpfully telling them that the kernel has blocked and is waiting for that page. 

Stopping the kernel in this way is useful if one is trying to take advantage of some sort of race condition or other issue. Assume, for example, that an attacker has identified a potential time-of-check-to-time-of-use vulnerability, where the ability to change a value in memory somewhere at the right time could cause the kernel to carry out some ill-advised action. Exploiting such a vulnerability requires hitting the window of time between when the kernel checks a value and when it acts on it; that window can be quite narrow. If the kernel can be made to block while that window is open, though, the attacker suddenly has all the time in the world. That can make a difficult exploit much easier. 

Attackers can be deprived of this useful tool by disallowing the handling in user space of faults incurred in kernel space. Simply changing the rules that way would almost certainly break existing code, though, so something else needs to be done. Colascione's patch addresses this problem in two steps, the first of which is to add a new flag (`UFFD_USER_MODE_ONLY`) for `userfaultfd()` which states that the resulting file descriptor can only be used for handling faults incurred in user space. Any descriptor created with this flag thus cannot be used for the sorts of attacks described above. 

One could try politely asking attackers to add `UFFD_USER_MODE_ONLY` to their `userfaultfd()` calls, but we are dealing with people who are not known for their observance of polite requests. So the patch set adds a new sysctl knob, concisely called `vm/unprivileged_userfaultfd_user_mode_only`, to make the request somewhat less polite; if it is set to one, `userfaultfd()` calls from unprivileged users will fail if that flag is not provided. At that point, kernel-space fault handling will no longer be available to attackers attempting to gain root access. The default value has to be zero, though, to maintain compatibility with older kernels. 

The only response to this patch set so far came from Peter Xu, who [pointed out](<https://lwn.net/ml/linux-kernel/20200506193816.GB228260@xz-x1/>) that the existing `vm/unprivileged_userfaultfd` knob could be extended instead. That knob can be used to disallow `userfaultfd()` entirely for unprivileged processes by setting it to zero, though its default value (one) allows such access. Xu suggested that setting it to two would allow unprivileged use, but for user-space faults only. This approach saves adding a new knob. 

Beyond that, the suggested change seems uncontroversial. It's a small patch that has no risk of breaking things for existing users, so there does not appear to be any real reason to keep it out.
