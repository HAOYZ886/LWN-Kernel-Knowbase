---
title: The kdbuswreck
url: https://lwn.net/Articles/641275/
date: "April 22, 2015"
category: Capabilities; kdbus
author: "By Jonathan Corbet April 22, 2015"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 22, 2015

Few readers will have failed to notice by now that the attempted merging of the kdbus interprocess communication system into the 4.1 kernel has failed to go as well as its proponents would have liked. As of this writing, the discussion continues and nothing has been merged. This article constitutes an attempt to derive a bit of light from the massive amounts of heat that 

Some corrections have been applied to this article; the old text remains and is ~~striked through~~. 

have been generated so far, with a specific focus on the issue of metadata and capabilities. 

Some observers have portrayed the opposition to kdbus as a front in the systemd wars, the intent being to obstruct its merging and set back the perceived systemd agenda. There have been a few messages mentioning systemd and expressing a lack of trust in its developers, but that has been the smallest part of the conversation; it can be safely disregarded. That is not where the serious objections come from. 

As was mentioned [last week](<https://lwn.net/Articles/640357/>), there is a certain level of discomfort with the core aspect of the design of kdbus: that it implements the D-Bus protocol. Some developers would rather not see kdbus in the kernel at all; others wish that it were an add-on to a more generic messaging solution. With regard to the D-Bus design, [this message from Havoc Pennington](<https://lwn.net/Articles/641277/>), one of the original designers of D-Bus, is worth a read. In short: he acknowledges that D-Bus is not perfect, but asserts that it does incorporate a lot of lessons from previous attempts and, as a result, it has been successful. 

The most specific advocate of a more general messaging solution is arguably Alan Cox. His latest [suggestion](<https://lwn.net/Articles/641278/>) would appear to be to go back to the old [AF_BUS approach](<https://lwn.net/Articles/504970/>); this patch implemented something D-Bus-like over sockets, but was rejected by the networking maintainers. Alan thinks it's worth another try, given that the kernel already has almost everything that is needed. There have been few signs, though, that the kdbus developers are in the mood to drop their work and attempt to resurrect an approach that has already failed once to get into the kernel. 

#### Metadata and capabilities

The fiercest bone of contention, though, would appear to be a topic that has come up before: the passing of process-specific metadata with messages. In particular, developers led by Andy Lutomirski have continued to assert that kdbus should not attach information about a sending process's capabilities and command line to messages as they cross the bus. 

The purpose of the transmission of capabilities, in particular, is to enable privileged processes on the bus to carry out actions at the request of another process on the bus — if that other process has the requisite capabilities. The plans for systemd involve allowing processes to request actions like changing the system time, tweaking the network configuration, or rebooting the system over the bus; the requested action will ~~only~~ be carried out if the requester has `CAP_SYS_TIME`, `CAP_NET_ADMIN`, or `CAP_SYS_BOOT`, respectively. 

The kdbus developers point out that one process can learn about another process's capabilities now by reading files in `/proc`. There's a little problem, though: reading from `/proc` is subject to race conditions. A process could request a privileged action over D-Bus, then quickly use `exec()` to run a setuid binary. If the `exec()` happens before the receiving process gets around to reading `/proc`, that process will see the new binary's elevated privileges and allow something that the original caller should not have been able to do. So capability-based authentication is not much used in current systems. One of the many appealing features of kdbus is that it makes such capability checks safe; the kernel can guarantee that the capabilities it transmits with the message are what the sending process held when the message was sent. 

Andy (and others) have a number of objections to this approach, starting with the ~~fact~~ assertion that capabilities are meant to be interpreted by the kernel, not by user space. By adding these features, user-space developers are said to be violating the layering of the system while broadening the meaning of the relevant capabilities — and they are generally seen as being overly broad already. As an example, `CAP_SYS_BOOT` gives the ability to call the `reboot()` system call and immediately reboot the system. Systemd will respond to a reboot request (from a process with `CAP_SYS_BOOT`) over D-Bus, however, by initiating a _clean_ reboot, unmounting filesystems, shutting down services, etc. Those are actions that `CAP_SYS_BOOT` would not enable on its own. Eric Biederman was quick to [suggest](<https://lwn.net/Articles/641281/>) that this extension of the `CAP_SYS_BOOT` capability could be helpful to an attacker. 

Andy also [pointed out](<https://lwn.net/Articles/641284/>) that the set of capabilities is determined by the kernel source. They can never be extended, so they will limit the expressiveness of authentication mechanisms using kdbus. It would be better, he said, to have a separate, capability-like mechanism implemented in user space that could be extended as the need for new privileges is encountered. 

Then there is an interesting little problem in the intersection of capabilities and [user namespaces](<https://lwn.net/Articles/532593/>). If a process connects to D-Bus, then moves into its own user namespace, it will appear to have all available capabilities. That would allow the capability checks to be bypassed entirely. This particular problem was fixed in kdbus some time ago by simply dropping the capability metadata when a message crosses a user-namespace boundary. But that fix comes at a cost: now the capability checks do not work at all for processes in user namespaces. The capability-based authentication mechanism, in other words, falls apart on a system where user namespaces are being used for containerization. Systemd maintainer Lennart Poettering [doesn't see this limitation as a problem](<https://lwn.net/Articles/641287/>) ~~because user namespaces are not (yet) heavily used~~ , but others may well disagree with this assessment. 

Eric [pointed out](<https://lwn.net/Articles/641417/>) that there is a capability translation mechanism that could be used to properly transmit capabilities across namespace boundaries. But he also complains that passing capabilities leaks information about sending processes and is thus a security problem in its own right. Linus was [not particularly sympathetic](<https://lwn.net/Articles/641418/>) to that particular concern, but others, [Andy](<https://lwn.net/Articles/641438/>) and [Alan](<https://lwn.net/Articles/641420/>) included, feel that a process should explicitly indicate that it intends to perform an action requiring a specific capability before any such information should be sent. 

Finally, though it hasn't been said explicitly, there is the simple fact that most kernel developers see capabilities as a failed experiment. There is no shortage of developers who would like to see them removed from the kernel altogether. That cannot be done — too many tiresome problems with applications breaking and such — but this feeling does lead to resistance to code that seems to expand the role of capabilities further. 

Lennart, though, maintains (in the message linked above) that capabilities do have their value and that capability checks are better than an all-or-nothing check for root privileges. He is not thrilled with the suggestion that kdbus should ~~support~~ implement a new user-space privilege mechanism, saying that ""we are not really in the business in designing comprehensive new access control systems that can be used for in-kernel and in-userspace subsystems."" There seems to be little inclination to consider alternatives (especially those that do not actually exist) at this point. 

And that seems to be the core of the impasse. Andy believes that this use of capabilities is dangerous, extending their meaning and bringing in a bunch of security-related code for little real benefit. The kdbus designers, instead, see metadata attachment as a useful tool for the implementation of sandboxing and privilege-separation schemes, and they are unwilling to drop it. Both positions seem firmly entrenched at this point, so it may well come down to what Linus decides to do. He has, for the most part, stayed out of the discussion, but in [one message](<https://lwn.net/Articles/641422/>) he indicated that most of the capability-related worries don't concern him that much. So he may yet pull kdbus into the kernel, though it would not be entirely surprising if it had to wait one more development cycle first.
