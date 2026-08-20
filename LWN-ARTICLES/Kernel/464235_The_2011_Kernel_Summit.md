---
title: The 2011 Kernel Summit
url: https://lwn.net/Articles/464235/
date: "October 24, 2011"
category: Kernel Summit
author: "By Jonathan Corbet October 24, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 24, 2011

The 2011 Kernel Summit was held in Prague on October 23-25. The organization of the event was changed somewhat this year; the first day was dedicated to a small number of minisummits. We do not currently have coverage from those events - that is a gap we hope to fill in the near future. 

#### Day 1

The Kernel Summit proper started on October 24 with the traditional closed session. The topics discussed there were: 

  * [Kernel.org report](<https://lwn.net/Articles/464233/>); including some more information about the compromise along with what is being done to secure this critical infrastructure going forward. 

  * [Tracing for large data centers](<https://lwn.net/Articles/464268/>) looked at Google's needs in the tracing area, but then the discussion moved into whether moving the tools into the kernel tree will help alleviate some of the problems with ABIs and backward compatibility. 

  * [Structured error logging](<https://lwn.net/Articles/464276/>); a presentation (only partly delivered) on improving kernel error logging. 

  * [Coming to love cgroups](<https://lwn.net/Articles/464288/>); while love for control groups may not have been the outcome, some more understanding of cgroups, controllers, and the problems with the latter did. 

  * [Memory management issues](<https://lwn.net/Articles/464292/>); a session on the progress the subsystem has made over the last year along with a list of patches that are in need of review and rework with an eye to eventually merging them. 

  * [Preemption disable and verifiable APIs](<https://lwn.net/Articles/464294/>) discussed calls to the `this_cpu*()` family as well as `preempt_disable()` calls multiplying throughout the kernel, which cause problems, and not just for the realtime kernel. 

  * [Scheduler testing](<https://lwn.net/Articles/464296/>); a reworked version of the LinSched scheduling simulator was discussed and proposed to be added to the kernel `tools/` directory. It may provide the long-sought ability to more reliably test scheduler changes. 

  * [Patch review](<https://lwn.net/Articles/464298/>); in a wide-ranging discussion, various problems in the area of patch review were covered. The discussion eventually turned to Android's kernel patches and, in particular, suspend blockers with a, perhaps, surprising conclusion. 

  * [Development process issues](<https://lwn.net/Articles/464302/>); there were fewer problems in this area than there have been in recent years. Linus is happy with how things are going, overall, though the growth of complexity in the kernel is somewhat worrisome. 

> [![\[Group Photo\]](https://static.lwn.net/images/conf/2011/lc-europe/KernelSummitGroup-sm.jpg)](<https://lwn.net/Articles/464307/>)

#### Day 2

The second day of the 2011 Kernel Summit featured a changed format: the session was open to all attendees. Coverage for this day has been split into two parts: 

  * [Morning](<https://lwn.net/Articles/464499/>): reports from a large number of minisummits, more on kernel.org security and the web of trust, and regression tracking. 

  * [Afternoon](<https://lwn.net/Articles/464500/>): shared libraries, failure handling, the media controller, the kernel build and configuration subsystem, and the future of the event itself. 

Needless to say, coverage would not be complete without a picture of the expanded group arranged into a rather compressed space: 

> [![\[Group photo\]](https://static.lwn.net/images/conf/2011/lc-europe/LargeKSGroup-sm.jpeg)](<https://lwn.net/Articles/464503/>)

(Your editor would like to thank the Linux Foundation for supporting his travel to Prague for this event).
