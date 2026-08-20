---
title: /dev/userfaultfd
url: https://lwn.net/Articles/897260/
date: "June 13, 2022"
category: userfaultfd
author: "By Jonathan Corbet June 13, 2022"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 13, 2022

The [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call allows one thread to handle page faults for another in user space. It has a number of interesting use cases, including the live migration of virtual machines. There are also some less appealing use cases, though, most of which are appreciated by attackers trying to take control of a machine. Attempts have been made over the years to make `userfaultfd()` less useful as an exploit tool, but [this patch set](<https://lwn.net/ml/linux-kernel/20220601210951.3916598-1-axelrasmussen@google.com/>) from Axel Rasmussen takes a different approach by circumventing the system call entirely. 

A call to `userfaultfd()` returns a special file descriptor attached to the current process. Among other things, this descriptor can be used (with [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl_userfaultfd.2.html>)) to register regions of memory. When any thread in the current process encounters a page fault in a registered area, it will be blocked and an event will be sent to the `userfaultfd()` file descriptor. The managing thread, on reading that event, has several options for how to resolve the fault; these include copying data into a new page, creating a zero-filled page, or mapping in a page that exists elsewhere in the address space. Once the fault has been dealt with, the faulting thread will continue its execution. 

A thread will normally encounter a page fault while running in user space; it may have dereferenced a pointer to a not-present page, for example. But there are times that page faults can happen within the kernel. As a simple example, consider a `read()` call; if the buffer provided to `read()` is not resident in RAM, a page fault will result when the kernel tries to access it. At that point, execution will be blocked as usual, but it will be blocked in the kernel rather than in user space. 

Blocking on page faults within the kernel is a normal experience when dealing with user-space memory, and everything works as it should. There is one little problem, though. If an attacker can force a page fault at a known point in the kernel — which is often not hard to do — they can use `userfaultfd()` to block the execution of a thread in the kernel indefinitely. That, in turn, can expand a race window that would otherwise be difficult or impossible to hit, giving the attacker a chance to change the world in potentially surprising ways while the kernel is waiting. 

This abuse of `userfaultfd()` is not just a theoretical possibility; various exploits ([example](<https://duasynt.com/blog/linux-kernel-heap-spray>)) using `userfaultfd()` have been disclosed over the years. The problem was deemed serious enough that [some restrictions](<https://lwn.net/Articles/819834/>) were added in 2020. If the `vm/unprivileged_userfaultfd` sysctl knob is set to zero (as it is on many distributions), then one of two conditions must apply for a `userfaultfd()` call to succeed: either the calling process has the `CAP_SYS_PTRACE` capability, or it supplies the `UFFD_USER_MODE_ONLY` flag to the system call. In the latter case, page faults encountered while running in the kernel will not be processed via the `userfaultfd()` mechanism, even if they occur within a registered area. 

This change was [merged for 5.11](<https://git.kernel.org/linus/37cd0575b851>) at the end of 2020. It closes off this use of `userfaultfd()` by attackers, but it also makes the full functionality unavailable to legitimate (but unprivileged) processes. As Rasmussen notes in [this patch](<https://lwn.net/ml/linux-kernel/20220601210951.3916598-3-axelrasmussen@google.com/>) from the series, that problem can be worked around by giving the process in question the `CAP_SYS_PTRACE` capability, but that enables a number of actions that have nothing to do with `userfaultfd()`. Specifically, it could allow the process to read data from or inject code into any other process on the system, which may be undesirable. It would be good, instead, to be able to enable the full `userfaultfd()` functionality for a process without granting it wider, unneeded privileges. 

Rasmussen's solution is to create a new special file called `/dev/userfaultfd` that gives access to this functionality without the need to call `userfaultfd()`. One might think that opening this file would yield a file descriptor that acts just like the descriptor from `userfaultfd()`, but it is not quite as simple. Instead, the only thing that can be done with a `/dev/userfaultfd` file descriptor is to call `ioctl()` with the `USERFAULTFD_IOC_NEW` command; _that_ will create a `userfaultfd()`-style file descriptor. 

A file descriptor created in this way will behave like one from `userfaultfd()` in every way, with one exception: the handling of kernel faults will be allowed regardless of the calling process's privilege level or the setting of the global sysctl knob. The effect, in other words, is to circumvent the 2020 patch, making full `userfaultfd()` features available again to all processes. The catch is that a process must be able to open `/dev/userfaultfd` in the first place to gain access to the feature it provides. By setting the access permissions on this file, an administrator can control who is able to open it and use `userfaultfd()` in this way. 

In other words, `/dev/userfaultfd` allows an administrator to give the ability to handle kernel faults to specific processes without the need to grant any other privileges. This patch series is in its third revision, and it would appear that the review comments received so far have been addressed. Barring some sort of surprise, this new tweak to the security policy surrounding `userfaultfd()` seems likely to find its way into the kernel during a near-future merge window.
