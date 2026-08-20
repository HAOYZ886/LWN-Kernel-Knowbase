---
title: The inherent fragility of seccomp()
url: https://lwn.net/Articles/738694/
date: "November 10, 2017"
category: "Security-seccomp"
author: "By Jonathan Corbet November 10, 2017"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 10, 2017

Kernel developers have worried for years that tracepoints could lead to applications depending on obscure implementation details; the consequent need to preserve existing behavior to avoid causing regressions could end up impeding future development. A recent report shows that the `seccomp()` system call is also more prone to regressions than users may expect — but kernel developers are unlikely to cause these regressions and, indeed, have little ability to prevent them. Programs using `seccomp()` will have an inherently higher risk of breaking when software is updated. 

[`seccomp()`](<http://man7.org/linux/man-pages/man2/seccomp.2.html>) allows the establishment of a filter that will restrict the set of system calls available to a process. It has obvious uses related to sandboxing; if an application has no need for, say, the `open()` system call, blocking access to that call entirely can reduce the damage that can be caused if that application is compromised. As more systems and programs are subjected to hardening, the use of `seccomp()` can be expected to continue to increase. 

Michael Kerrisk recently [reported](<https://lwn.net/Articles/738695/>) that upgrading to glibc 2.26 broke one of his demonstration applications. That program was using `seccomp()` to block access to the `open()` system call. The problem that he ran into comes down to the fact that applications almost never invoke system calls directly; instead, they call wrappers that have been defined by the C library. 

The glibc `open()` wrapper has, since the beginning, been a wrapper around the kernel's `open()` system call. But `open()` is an old interface that has long been superseded by `[openat()](<http://man7.org/linux/man-pages/man2/openat.2.html>)`. The older call still exists because applications expect it to be there, but it is implemented as a special case of `openat()` within the kernel itself. In glibc 2.26, the `open()` wrapper was changed to call `openat()` instead. This change was not visible to ordinary applications, but it will break `seccomp()` filters that behave differently for `open()` and `openat()`. 

Kerrisk was not really complaining about the change, but he did want to inform the glibc developers that there were user-visible effects from it: ""I want to raise awareness that these sorts of changes have the potential to possibly cause breakages for some code using seccomp, and note that I think such changes should not be made lightly or gratuitously"". The developers should, he was suggesting, keep the possibility of breaking `seccomp()` filters in mind when making changes, and they should document such changes when they cannot be avoided. 

Florian Weimer, however, [disagreed](<https://lwn.net/Articles/738696/>): 

I have the opposite view: We should make such changes as often as possible, to remind people that seccomp filters (and certain SELinux and AppArmor policies) are incompatible with the GNU/Linux model, where everything is developed separately and not maintained within a single source tree (unlike say OpenBSD). This means that you really can't deviate from the upstream Linux userspace ABI (in the broadest possible sense) and still expect things to work. 

Another way of putting this might be: `seccomp()` filters are not considered to be a part of the ABI that is provided by glibc, so incompatible changes there are not considered regressions. They are, instead, a consequence of filtering below the glibc level while expecting behavior above that level to remain unchanged. 

Weimer's point of view would appear to be the one that will govern glibc development going forward. So Kerrisk has [proposed some man-page changes](<https://lwn.net/Articles/738697/>) to make the fragility of `seccomp()` filters a bit less surprising to developers. Playing the game at this level will require a fairly deep understanding of what is going on and the ability to adapt to future C-library changes. 

This outcome could be seen as an argument in favor of a filtering interface like OpenBSD's [`pledge()`](<http://man.openbsd.org/cgi-bin/man.cgi/OpenBSD-current/man2/pledge.2>). Like `seccomp()`, `pledge()` is used to limit the set of system calls available to a process, but `pledge()` is defined in terms of broad swathes of functionality rather than working at the level of individual system calls. It can be used to allow basic file I/O, for example, while disabling the opening (or creation) of new files. `pledge()` is far less expressive than `seccomp()` and cannot implement anything close to the same range of policies but, for basic filtering, it seems far less likely to generate surprises after a kernel or library update. 

But Linux doesn't have `pledge()` and seems unlikely to get it. `seccomp()` can certainly get the sandboxing job done, but developers who use it should expect to spend some ongoing effort maintaining their filters.
