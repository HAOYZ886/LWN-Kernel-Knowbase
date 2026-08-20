---
title: "How far do we want to go with BPF?"
url: https://lwn.net/Articles/908464/
date: "September 19, 2022"
category: BPF
author: "By Jonathan Corbet September 19, 2022 Maintainers Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 19, 2022

* * *

[Maintainers Summit](<https://lwn.net/Articles/908320/>)

The BPF subsystem has come a long way in recent years; what started as a mechanism for implementing packet filters has become a way to load code into the kernel for a wide variety of tasks. At the 2022 Linux Kernel Maintainers Summit, Jiri Kosina kicked off a session by asking how far the transition to BPF should go. The actual scope of the session turned out to be rather more limited than that, and no fundamental changes were considered. 

Kosina started by saying that BPF has been highly successful and it is increasingly being used by user programs. But the kernel community has no policies about out-of-tree BPF code and, in particular, how to deal with associated bug reports. What happens when a kernel change breaks somebody's BPF program? Ted Ts'o added that future applications are increasingly likely to include BPF scripts, and users may not be aware that they are running something that is more like a kernel module than an ordinary program. That can lead to confusion and complaints to kernel developers when things break. 

[![\[Jiri Kosina\]](https://static.lwn.net/images/conf/2022/lpc/JiriKosina-sm.png)](<https://lwn.net/Articles/908476/>) Linus Torvalds did his best to cut this conversation short; it was, he said, the same as the discussions about the tracepoint API that have been had multiple times over the years. People worry about problems caused by internal kernel changes, but he has never seen such a problem in practice. A BPF program that depends on kernel symbols, he said, is not really a "user program" anymore; "only Facebook people with system-management tools" are running such things. Steve Rostedt pointed out that systemd loads BPF programs now; Torvalds answered that it is working, there have been no complaints, and he did not want to waste time worrying about theoretical problems. 

I couldn't resist raising a related issue. The kernel community looks closely at symbols exported to loadable modules, but symbols exported to BPF programs (as "[kfuncs](<https://lwn.net/Articles/856005/>)") are harder to see and tend to go under the radar. Christoph Hellwig suggested that a new macro for kfuncs should be introduced so that these exports can at least be found with `grep`. 

I also said that BPF changes tend to be invisible to the rest of the kernel community since they go into the mainline as part of the networking pull requests, which can include thousands of non-BPF commits. It might be better, I suggested, if BPF stopped hiding inside the networking subsystem. BPF goes far beyond networking these days, and it would make sense for BPF changes to go directly to Torvalds via their own pull requests. Networking maintainer Jakub Kicinski responded that working that way would be painful since there is still a lot of work that crosses the BPF and networking trees; BPF maintainer Alexei Starovoitov agreed. It seems likely that nothing will change in how those trees are managed. 

At that point, any appetite for BPF discussions appeared to have run its course and this session came to a close.
