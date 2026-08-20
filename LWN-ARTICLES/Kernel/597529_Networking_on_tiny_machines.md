---
title: Networking on tiny machines
url: https://lwn.net/Articles/597529/
date: "May 7, 2014"
category: Embedded systems; Networking
author: "By Jonathan Corbet May 7, 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 7, 2014

[Last week's article](<https://lwn.net/Articles/596754/>) on "Linux and the Internet of Things" discussed the challenge of shrinking the kernel to fit on to computers that, by contemporary standards, are laughably underprovisioned. Shortly thereafter, the posting of a kernel-shrinking patch set sparked a related discussion: what needs to be done to get the kernel to fit into tiny systems and, more importantly, is that something that the kernel development community wants to even attempt? 

#### Shrinking the network stack

The patch set in question was [a 24-part series](<https://lwn.net/Articles/597398/>) from Andi Kleen adding an option to build a minimally sized networking subsystem. Andi is looking at running Linux on systems with as little as 2MB of memory installed; on such systems, the Linux kernel's networking stack, which weighs in at about 400KB for basic IPv4 support, is just too big to shoehorn in comfortably. By removing a lot of features, changing some data structures, and relying on the [link-time optimization](<https://lwn.net/Articles/512548/>) feature to remove the (now) unneeded code, Andi was able to trim things down to about 170KB. That seems like a useful reduction, but, as we will see, these changes have a rough road indeed ahead of them before any potential merge into the mainline. 

Some of the changes in Andi's patch set include: 

  * Removal of the "ping socket" feature that allows a non-setuid `ping` utility to send ICMP echo packets. It's a useful feature in a general-purpose distribution, but it's possibly less useful in a single-purpose tiny machine that may not even have a `ping` binary. Nonetheless the change was [rejected](<https://lwn.net/Articles/597532/>): ""We want to move away from raw sockets, and making this optional is not going to help us move forward down that path"". 

  * Removal of raw sockets, saving about 5KB of space. [Rejected](<https://lwn.net/Articles/597533/>): ""Sorry, you can't have half a functioning ipv4 stack."" 

  * Removal of the [TCP fast open](<https://lwn.net/Articles/508865/>) feature. That feature takes about 3KB to implement, but it also requires the kernel to have the crypto subsystem and AES code built in. [Rejected](<https://lwn.net/Articles/597534/>): ""It's for the sake of the remote service not the local client, sorry I'm not applying this, it's a facility we want to be ubiquitous and in widespread use on as many systems as possible."" 

  * Removal of the BPF packet filtering subsystem. [Rejected](<https://lwn.net/Articles/597535/>): ""I think you highly underestimate how much 'small systems' use packet capturing and thus BPF."" 

  * Removal of the MIB statistics collection code (normally accessed via `/proc`) when `/proc` is configured out of the kernel. [Rejected](<https://lwn.net/Articles/597548/>): ""Congratulations, you just broke ipv6 device address netlink dumps amongst other things"". 

The above list could be made much longer, but the point should be apparent by now: this patch set was not welcomed by the networking community with open arms. This community has been working with a strong focus on performance and features on contemporary hardware; networking developers (some of them, at least) do not want to be bothered with the challenges of trying to accommodate users of tiny systems. As Eric Dumazet [put it](<https://lwn.net/Articles/597537/>): 

I have started using linux on 386/486 pcs which had more than 2MB of memory, it makes me sad we want linux-3.16 to run on this kind of hardware, and consuming time to save few KB here and here. 

The networking developers also do not want to start getting bug reports from users of a highly pared-down networking stack wondering why things don't work anymore. Some of that would certainly happen if a patch set like this one were to be merged. One can try to imagine which features are absolutely necessary and which are optional on tiny systems, but other users solving different problems will come to different conclusions. A single "make it tiny" option has a significant chance of providing a network stack with 99% of what most tiny-system users need — but the missing 1% will be different for each of those users. 

#### Should we even try?

Still, pointing out some difficulties inherent in this task is different from saying that the kernel should not try to support small systems at all, but that appears to be the message coming from the networking community. At one point in the discussion, Andi [posed a direct question](<https://lwn.net/Articles/597538/>) to networking maintainer David Miller: ""What parts would you remove to get the foot print down for a 2MB single purpose machine?"" David's [answer](<https://lwn.net/Articles/597539/>) was simple: ""I wouldn't use Linux, end of story. Maybe two decades ago, but not now, those days are over."" In other words, from his point of view, Linux should not even try to run on machines of that class; instead, some sort of specialty operating system should be used. 

That position may come as a bit of a surprise to many longtime observers of the Linux development community. As a general rule, kernel developers have tried to make the system work on just about any kind of hardware available. The "go away and run something else" answer has, on rare occasion, been heard with regard to severely proprietary and locked-down hardware, but, even in those cases, somebody usually makes it work with Linux. In this case, though, there is a class of hardware that _could_ run Linux, with users who would _like_ to run Linux, but some kernel developers are telling them that there is no interest in adding support for them. This is not a message that is likely to be welcomed in those quarters. 

Once upon a time, vendors of mainframes laughed at minicomputers — until many of their customers jumped over to the minicomputer market. Minicomputer manufacturers treated workstations, personal computers, and Unix as toys; few of those companies are with us now. Many of us remember how the proprietary Unix world treated Linux in the early days: they dismissed it as an underpowered toy, not to be taken seriously. Suffice to say that we don't hear much from proprietary Unix now. It's a classic [Innovator's Dilemma](<http://en.wikipedia.org/wiki/The_Innovator%27s_Dilemma>) story of disruptive technologies sneaking up on incumbents and eating their lunch. 

It is not entirely clear that microscopic systems represent this type of disruptive technology; the "wait for the hardware to grow up a bit" approach has often worked well for Linux in the past. It is usually safe to bet on computing hardware increasing in capability over time, so effort put into supporting underpowered systems is often not worth it. But we may be dealing with a different class of hardware here, one where "smaller and cheaper" is more important than "more powerful." If these systems can be manufactured in vast numbers and spread like "smart dust," they may well become a significant part of the computing substrate of the future. 

So the possibility that tiny systems could be a threat to Linux should certainly be considered. If Linux is not running on those devices, something else will be. Perhaps it will be a Linux kernel with the networking stack replaced entirely by a user-space stack like [lwIP](<http://savannah.nongnu.org/projects/lwip/>), or perhaps it will be some other free operating system whose community is more interested in supporting this hardware. Or, possibly, it could be something proprietary and unpleasant. However things go, it would be sad to look back someday and realize that the developers of Linux could have made the kernel run on an important class of machines, but they chose not to.
