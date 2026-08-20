---
title: CAP_PERFMON — and new capabilities in general
url: https://lwn.net/Articles/812719/
date: "February 21, 2020"
category: Capabilities
author: "By Jonathan Corbet February 21, 2020"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 21, 2020

The `[perf_event_open()](<http://man7.org/linux/man-pages/man2/perf_event_open.2.html>)` system call is a complicated beast, requiring a fair amount of study to master. This call also has some interesting security implications: it can be used to obtain a lot of information about the running system, and the complexity of the underlying implementation has made it more than usually prone to unpleasant bugs. In current kernels, the security controls around `perf_event_open()` are simple, though: if you have the `CAP_SYS_ADMIN` capability, `perf_event_open()` is available to you (though the system administrator can make it available without any privilege at all). Some current work to create a new capability for the perf events subsystem would seem to make sense, raising the question of why adding new capabilities isn't done more often. 

Capabilities are a longstanding effort to split apart the traditional Unix superuser's powers into something more fine-grained, allowing administrators to give limited privileges where needed without making the recipients into full superusers. There are [37 capabilities](<http://man7.org/linux/man-pages/man7/capabilities.7.html>) defined in current Linux kernels, controlling the ability to carry out a range of tasks including configuring terminal devices, overriding resource limits, installing kernel modules, or adjusting the system time. Among these capabilities, though, is `CAP_SYS_ADMIN`, nominally the capability needed to perform system-administration tasks. `CAP_SYS_ADMIN` has become the default capability to require when nothing else seems to fit; it enables so many actions that it has long been known as "[the new root](<https://lwn.net/Articles/486306/>)". 

A quick check shows well over 500 checks for `CAP_SYS_ADMIN` in the 5.6-rc kernel. During the 5.6 merge window, new checks were added to allow holders of `CAP_SYS_ADMIN` to send hardware-specific commands to obscure devices, configure [time namespaces](<https://lwn.net/Articles/766089/>), load [BPF programs for kernel operations structures](<https://lwn.net/Articles/811631/>), and open access to x86 [MTRR](<https://en.wikipedia.org/wiki/Memory_type_range_register>) registers. The perf events subsystem has also come to rely on `CAP_SYS_ADMIN` to keep unprivileged users out. As a result, to enable a user to call `perf_event_open()`, an administrator must also allow that user to mount filesystems, access PCI configuration spaces, tune memory-management policies, load BPF programs, and more. That is a lot of privilege to associate with a task that ordinary users are fairly likely to legitimately need to do. 

This [patch set](<https://lwn.net/ml/linux-kernel/c8de937a-0b3a-7147-f5ef-69f467e87a13@linux.intel.com/>) from Alexey Budankov addresses that problem by creating a new capability called `CAP_PERFMON` to govern performance-monitoring tasks. With this patch installed, users (or their programs) could be granted `CAP_PERFMON` rather than `CAP_SYS_ADMIN`, enabling them to get performance data without adding all those other powers. Of course, `CAP_SYS_ADMIN` would still be sufficient to call `perf_event_open()`; otherwise the chances of breaking existing systems are high. But it would no longer be necessary if a user has `CAP_PERFMON` instead. 

At a first look, this change seems relatively obvious; it is hard to complain about separating out a relatively constrained, low-danger activity from a powerful capability. But it does lead one to wonder why this kind of change is done so rarely. The last time a new capability was added was in 2014, when [`CAP_AUDIT_READ`](<https://git.kernel.org/linus/3a101b8de0d3>) joined the set. It would appear that the last time a capability was split out of `CAP_SYS_ADMIN` was [the creation of `CAP_SYSLOG`](<https://git.kernel.org/linus/ce6ada35bdf7>) in 2010. Once something becomes part of `CAP_SYS_ADMIN`, it seems, it stays there. Why might that be the case? 

One reason, of course, is the aforementioned compatibility issue: once `CAP_SYS_ADMIN` allows an action, it can never lose that power without possibly breaking existing systems. When Serge Hallyn added `CAP_SYSLOG`, he added the usual code that made things continue to work if the process in question had `CAP_SYS_ADMIN`. In that case, though, the kernel issues a warning that use of `CAP_SYS_ADMIN` for these operations is deprecated. Nearly ten years later, the compatibility code — and the warning — remain. Splitting capabilities out of `CAP_SYS_ADMIN` is less than fully rewarding when the power of `CAP_SYS_ADMIN` itself can never be reduced. 

Adding capabilities has hazards of its own, in that existing code will know nothing about a new capability and what it might control. A program that clears bits out of a capability mask is likely to clear the new one, but that capability might be needed going forward. Experience has shown that running a privileged program with selectively removed capabilities can open up surprising vulnerabilities; every new capability potentially creates just that sort of situation. So capabilities must be added with care. There is a reason why the SELinux build has [a check that explicitly fails](<https://elixir.bootlin.com/linux/latest/source/security/selinux/include/classmap.h#L31>) if new capabilities have been added without corresponding changes in SELinux itself. 

Then, there is the unfortunate fact that capabilities in Linux are seen by many as a failed experiment. Nobody has ever made a practical, fully capability-based system using them, and many of the defined capabilities are relatively easily escalated to full root powers. Linux systems above the kernel level have made limited use of them, if indeed capabilities have been used at all. It can be hard to generate enthusiasm for refining a system that can never work as was originally intended and which may never be used in any serious way. 

As an example, one obvious way to use capabilities to reduce privilege would be to remove the setuid bit on existing utilities and install just the needed capabilities instead. The kernel has supported [file-based capabilities](<https://lwn.net/Articles/211883/>) since the 2.6.24 release in 2008 after all. Your editor's current system, running Fedora 31 (which includes "first" among its goals) contains a grand total of nine binaries with capabilities attached: 

```
# getcap -r /
        /usr/bin/gnome-keyring-daemon = cap_ipc_lock+ep
        /usr/bin/clockdiff = cap_net_raw+p
        /usr/bin/arping = cap_net_raw+p
        /usr/bin/newuidmap = cap_setuid+ep
        /usr/bin/newgidmap = cap_setgid+ep
        /usr/bin/ping = cap_net_admin,cap_net_raw+p
        /usr/bin/gnome-shell = cap_sys_nice+ep
        /usr/sbin/mtr-packet = cap_net_raw+ep
        /usr/sbin/suexec = cap_setgid,cap_setuid+ep
```

It is good to know that `gnome-shell` does not run setuid root, so capabilities have brought some value here. But that compares with 31 setuid root binaries; it would appear that there is no prospect of this distribution becoming capability-only anytime soon. 

That said, there are signs of a shift with regard to capabilities. The never-ending desire to harden our systems against attacks is driving developers to take another look at Linux capabilities and how they might help. The Android system makes use of capabilities, for example. Systemd gives administrators extensive control over the capabilities granted to running programs. It may just be that, after many years of disuse, Linux capabilities are finally finding a place in deployed systems. 

If that is the case, we may well see a renewed level of interest in increasing the granularity of the permissions controlled by capabilities. That could include splitting more powers out of `CAP_SYS_ADMIN` though, as noted above, that must be done carefully. `CAP_SYS_ADMIN` is unlikely to stop being the not-so-new root anytime soon, but perhaps it could be made into a capability that few programs need to have to get their work done.
