---
title: Debating the value of XDP
url: https://lwn.net/Articles/708087/
date: "December 6, 2016"
category: "Networking-eXpress Data Path XDP"
author: "By Jonathan Corbet December 6, 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 6, 2016

All parts of the kernel are shaped by the need for performance and scalability, but some subsystems feel those pressures more than others. The networking subsystem, which can be asked to handle steady-state workloads involving millions of packets per second, has had to look at numerous technologies in its search for improved performance. One result of that work has been the "express data path" (or XDP) mechanism. Now, however, XDP is seeing some pushback from developers who see it as "pointless," a possible long-term maintenance problem, and a distraction from the work that networking developers need to be doing. 

The core idea behind XDP is optimizing cases where relatively simple decisions can be made about incoming packets. It allows the loading of a BPF program into the kernel; that program gets an opportunity to inspect packets before they enter the networking stack itself. The [initial use case](<https://lwn.net/Articles/682538/>) for XDP was to enable the quick dropping of unwanted packets, but it has since expanded to cover simple routing decisions and packet modifications; see [this in-progress documentation](<https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/index.html>) for some more information on how it works. 

The core benefit of XDP is that the system can make quick decisions about packets without the need to involve the rest of the networking code. Performance could possibly be further improved in some settings by loading XDP programs directly into the network interface, perhaps after a translation step. 

Thus far, most of the public discussion about XDP has been focused on the details of its implementation rather than on whether XDP is a good idea in the first place. That came to an end at the beginning of December, though, when Florian Westphal, in [a posting](<https://lwn.net/Articles/708136/>) written with help from Hannes Frederic Sowa, let it be known that he disagrees: ""Lots of XDP related patches started to appear on netdev. I'd prefer if it would stop..."" He would rather that developers turned away from a ""well meaning but pointless"" approach toward something that, in his view, is better suited to the problems faced by the networking subsystem. 

That something, in short, is any of the mechanisms out there (such as the [data plane development kit](<http://dpdk.org/>)) that allow the networking stack to be bypassed by user-space code. These mechanisms can indeed yield improved performance in settings where a strictly defined set of functionality is needed and the benefits that come from a general-purpose network stack can be done without. Additionally, he said, some problems are best solved by utilizing the packet-filtering features implemented in the hardware. 

XDP, Westphal said, is an inferior solution because it provides a poorer programming environment. Networking code done in user space can be written in any of a range of languages, has full debugging support available, and so on. BPF programs, instead, are harder to develop and much more limited in their potential functionality. Looking at a number of use cases for XDP, including routing, load balancing, and early packet filtering, he claims that there are better solutions for each. 

Thomas Graf [responded](<https://lwn.net/Articles/708144/>) that he has a fundamental problem with user-space networking: as soon as a packet leaves the kernel, anything can happen to it and it is no longer possible to make security decisions about it. User-space code could be compromised and there is no way for the kernel to know. BPF code in the kernel, instead, should be more difficult to compromise since its freedom of action is much more restricted. He also said that load balancing often needs to be done within applications as well as across machines, and he would not want to see that done in user space. 

Sowa, instead, [questioned](<https://lwn.net/Articles/708145/>) the early-drop use case, asking whether the focus was on additional types of protection or improved performance. Like Westphal, he suggested that this problem could be solved primarily with hardware-based packet dropping. The [answer](<https://lwn.net/Articles/708146/>) from Tom Herbert made it clear that he sees both flexibility and performance as being needed: 

DDOS mitigation alone is probably a sufficient motivation to look at XDP. We need something that drops bad packets as quickly as possible when under attack, we need this to be integrated into the stack, we need it to be programmable to deal with the increasing savvy of attackers, and we don't want to be forced to be dependent on HW solutions. This is why we created XDP! 

Networking maintainer David Miller also [said](<https://lwn.net/Articles/708147/>) that he sees XDP as being a good solution for packet dropping use cases. Hardware-based filters, he said, are not up to the job, and XDP looks to him to be the right approach. 

Sowa's other concern was not so easily addressed, though. As XDP programs gain functionality, they will need access to increasingly sophisticated information from the rest of the networking stack. That information can be provided by way of functions callable from BPF, but those functions will likely become part of the kernel's user-space ABI. That, in turn, will limit the changes that the networking developers can make in the future. These concerns mirror the [worries about tracepoints](<https://lwn.net/Articles/705270/>) that have limited their use in parts of the kernel. Nobody in the discussion addressed the ABI problem; in the end, it will have to be handled like any other user-space interface, where new features are, hopefully, added with care and a lot of review. 

In the end, the discussion probably changed few minds about the value of XDP — or the lack thereof. Stephen Hemminger probably summarized things best when he [said](<https://lwn.net/Articles/708153/>) that there is room for a number of different approaches. XDP is better for ""high speed packet mangling"", while user-space approaches are going to be better for the implementation of large chunks of networking infrastructure. The networking world is complex, and getting more so; a variety of approaches will be needed to solve all the problems that the kernel is facing in this area.
