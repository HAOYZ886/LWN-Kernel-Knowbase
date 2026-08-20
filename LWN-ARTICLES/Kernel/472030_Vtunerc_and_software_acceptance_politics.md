---
title: Vtunerc and software acceptance politics
url: https://lwn.net/Articles/472030/
date: "December 14, 2011"
category: "Development model-Driver merging"
author: "By Jonathan Corbet December 14, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 14, 2011

The kernel development process prides itself on being driven exclusively by technical concerns. Ideally, all decisions with regard to the merging of code would be based on whether that code makes technical sense or not; decisions based on "political" concerns are seen as being rather less ideal. But, as a recent discussion shows, even a seemingly "political" decision can have technical reasoning behind it. 

In June 2011, Honza Petrous posted [a patch](<http://www.spinics.net/lists/linux-media/msg34240.html>) to the linux-media list containing an implementation of a virtual DVB (digital video broadcast) device driver. DVB drivers normally talk to devices that tune in and capture video streams - television tuners, in other words. Honza's "vtunerc" driver, instead, drives no physical hardware at all. Instead, it serves as a sort of loopback device. One side looks like a normal DVB device; it handles all the usual DVB system calls. The other side, which shows up as `/dev/vtunerc _N_` , passes a processed form of those DVB system calls back to user space. The intended use is for a user-space process to receive those operations and pass them to a remote peer elsewhere on the network; that peer would then perform the operations on a real DVB device. Using this mechanism, DVB devices could be hosted on a network in a manner that is entirely transparent to DVB applications. Honza has posted [a diagram](<http://www.nessiedvb.org/wiki/doku.php?id=vtuner_bigpicture>) showing how the pieces fit together. 

Virtual devices of this type are not unprecedented in the Linux (and Unix) tradition; the venerable virtual terminal devices work in much the same way. This type of mechanism is also sometimes used to make devices available within virtualized guest systems. But this patch was not accepted into the DVB subsystem for a number of reasons, one of which being that it would facilitate the creation of proprietary user-space drivers for DVB devices. That was the reason Honza picked up on when he [went to the linux-kernel list](<https://lwn.net/Articles/472038/>) in an attempt to gain support in November, saying that, while he didn't discount the possibility of "bad guys" abusing the interface to create closed-source drivers, he was not convinced that it justified the "aggressive NACK" the code received. 

As the subsequent discussion made clear, some developers do, indeed, believe that the potential for abuse in this way is sufficient reason to keep an interface out of the mainline kernel. That is the same reasoning that has, for example, blocked the merging of graphics drivers that have proprietary user-space components. But it also turns out that there is rather more than that to this particular decision. Reasons for keeping vtunerc out include: 

  * The same ABI that enables proprietary drivers also exposes a fair amount of internal information about the DVB layer. That ABI would have to remain unchanged even as DVB evolves, leading to maintenance burdens in the future. 

  * There appears to be little advantage to routing all that video data through the kernel and immediately back to user space; it would make more sense for DVB applications to use a network video protocol directly and avoid the cost of routing data through the kernel. 

  * DVB applications tend to work with tight timing constraints. Adding a network connection into the mix will create latencies that may well confuse these applications. Working across a network requires a different approach than talking to a device directly; operations that may be done synchronously on a local device may need to happen asynchronously with a remote device. By hiding the network link, vtunerc makes it impossible for applications to drive the device appropriately. 

  * If the creation of this type of loopback device absolutely cannot be avoided, it can be done with the [CUSE](<https://lwn.net/Articles/308445/>) (char drivers in user space) interface instead of adding a new ABI. 

In the discussion, it seems that much of the motivation for vtunerc comes from the fact that it would require no changes to applications at all, while a user-space approach might require such changes. In fact, it [seems that there is a political problem at that level](<https://lwn.net/Articles/472044/>): the maintainer of the Video Disk Recorder (VDR) tool is evidently uninterested in adding real network client support. Needless to say, adding an interface to the kernel to get around an uncooperative application maintainer is not an idea that gains a lot of sympathy on the kernel side. 

It is easy to see politics in decisions that do not go one's way. As the old saying goes: just because you're paranoid doesn't mean that they aren't out to get you; in some cases non-technical agendas almost certainly play a part. But it may also be that the proposed code simply is not acceptable in its current form and needs work. Going back to the mailing lists and crying "politics" is an almost certain way to turn it into a political issue, though, and with an almost certainly undesirable result.
