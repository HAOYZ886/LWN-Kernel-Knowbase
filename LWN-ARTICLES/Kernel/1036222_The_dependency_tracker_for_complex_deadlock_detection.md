---
title: The dependency tracker for complex deadlock detection
url: https://lwn.net/Articles/1036222/
date: "September 4, 2025"
category: Development tools
author: "By Jonathan Corbet September 4, 2025 OSS EU"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 4, 2025

* * *

[OSS EU](<https://lwn.net/Archives/ConferenceByYear/#2025-Open_Source_Summit_Europe>)

Deadlocks are a constant threat in concurrent settings with shared data; it is thus not surprising that the kernel project has long since developed tools to detect potential deadlocks so they can be fixed before they affect production users. Byungchul Park thinks that he has developed a better tool that can detect more deadlock-prone situations. At the 2025 [Open Source Summit Europe](<https://events.linuxfoundation.org/open-source-summit-europe/>), he presented an introduction to his dependency tracker (or "DEPT") tool and the kinds of problems it can detect. 

[![\[Byungchul Park\]](https://static.lwn.net/images/conf/2025/osseu/ByungchulPark-sm.png)](<https://lwn.net/Articles/1036238/>) Park began by presenting a simple ABBA deadlock scenario. Imagine two threads running, each of which makes use of two locks, called `A` and `B`. The first thread acquires `A`, then `B`, ending up holding both locks; meanwhile, the second thread acquires the same two locks but in the opposite order, taking `B` first. That can work, until the bad day when each thread succeeds in taking the first of its two locks. Then one holds `A`, the other holds `B`, and each is waiting for the other to release the second lock it needs. They will wait for a long time. 

Deadlocks like this can be hard to reproduce, and thus hard to track down and fix; it is far better to detect the possibility of such a deadlock ahead of time. The kernel has a tool called "lockdep", originally written by Ingo Molnar, that can perform this detection. Whenever a thread acquires a lock, lockdep remembers that acquisition, along with the context — specifically, any other locks that are currently held. It will thus notice that, in the first case, `A` is acquired ahead of `B`. When lockdep observes the second thread acquiring the locks in the opposite order, it will raise the alarm. The problem can then be fixed, hopefully before this particular deadlock ever occurs on a production system. 

Park continued with a more complex example, presented in this form:
