---
title: Memory management for virtualization
url: https://lwn.net/Articles/382299/
date: "April 7, 2010"
category: "Huge pages; Memory management-Virtualization; Virtualization"
author: "By Jonathan Corbet April 7, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 7, 2010

For some time now, your editor has asserted that, at the kernel level, the virtualization problem is mostly solved. Much of the remaining work is in the performance area. That said, making virtualized systems perform well is not a small or trivial problem. One of the most interesting aspects of this problem is in the interaction between virtualized guests and host memory management. A couple of patch sets under discussion illustrate where the work in this area is being done. 

The [transparent huge pages patch set](<http://lwn.net/Articles/359158/>) was discussed here back in October. This patch seeks to change how huge pages are used by Linux applications. Most current huge page users must be set up explicitly to use huge pages, which, in turn, must be set aside by the system administrator ahead of time; see [the recent series by Mel Gorman](<http://lwn.net/Articles/374424/>) for more information on how this is done. The "some assembly required" nature of huge pages limits their use in many situations. 

The transparent huge page patch, instead, works to provide huge pages to applications without those applications even being aware that such pages exist. When large pages are available, applications may have their scattered pages joined together into huge pages automatically; those pages can also be split back apart when the need arises. When the system operates in this mode, huge pages can be used in many more situations without the need for application or administrator awareness. This feature turns out to be especially beneficial when running virtualized guests; huge pages map well to how guests tend to see and use their address spaces. 

The transparent huge page patches have been working their way toward acceptance, though it should be noted that some developers still have complaints about this work. Andrew Morton recently [pointed out a different problem](<https://lwn.net/Articles/382303/>) with this patch set: 

It appears that these patches have only been sent to linux-mm. Linus doesn't read linux-mm and has never seen them. I do think we should get things squared away with him regarding the overall intent and implementation approach before trying to go further... [T]his is a *large* patchset, and it plays in an area where Linus is known to have, err, opinions. 

It didn't take long for Linus to join the conversation directly; after a couple of digressions into areas not directly related to the benefits of the transparent huge pages patch, he realized that this work was motivated by the needs of virtualization. At that point, he [lost interest](<https://lwn.net/Articles/382307/>): 

So I thought it was a more interesting load than it was. The virtualization "TLB miss is expensive" load I can't find it in myself to care about. "Get a better CPU" is my answer to that one. 

He [went on](<https://lwn.net/Articles/382308/>) to compare the transparent huge page work to high memory, which, in turn, he called ""a failure"". The right solution in both cases, he says, is to get a better CPU. 

It should be pointed out that high memory was a spectacularly successful failure, extending the useful life of 32-bit systems for some years. It still shows up in surprising places - you editor's phone is running a high-memory-enabled kernel. So calling high memory a failure is something like calling the floppy driver a failure; it may see little use now, but there was a time when we were glad we had it. 

Perhaps, someday, advances in processor architecture will make transparent huge pages unnecessary as well. But, while the alternative to high memory (64-bit processors) has been in view for a long time, it's not at all clear what sort of processor advance might make transparent huge pages irrelevant. So, should this code get into the kernel, it may well become one of those failures which is heavily used for many years. 

A related topic under discussion was the recently-posted [VMware balloon driver](<https://lwn.net/Articles/382314/>) patch. A balloon driver has an interesting task; its job is to "inflate" within a guest system, taking up memory and making it unavailable for processes running within the guest. The pages absorbed by the balloon can then be released back to the host system which, presumably, has a more pressing need for them elsewhere. Letting "air" out of the balloon makes memory available to the guest once again. 

The purpose of this driver, clearly, is to allow the host to dynamically balance the memory needs of its guest systems. It's a bit of a blunt instrument, but it's the best we have. But Andrew Morton [questioned](<https://lwn.net/Articles/382316/>) the need for a separate memory control mechanism. The kernel already has a function, called `shrink_all_memory()`, which can be used to force the release of memory. This function is currently used for hibernation, but Andrew suspects that it could be adapted to the needs of virtualization as well. 

Whether that is really true remains to be seen; it seems that the bulk of the complexity lies not with the freeing of memory but in the communication between the guest and the hypervisor. Beyond that, the longer-term solution is likely to be something more sophisticated than simply applying memory pressure and watching the guest squirm until it releases enough pages. As Dan Magenheimer [put it](<https://lwn.net/Articles/382318/>): 

Historically, all OS's had a (relatively) fixed amount of memory and, since it was fixed in size, there was no sense wasting any of it. In a virtualized world, OS's should be trained to be much more flexible as one virtual machine's "waste" could/should be another virtual machine's "want". 

His answer to this problem is the [transcendent memory patch](<http://lwn.net/Articles/340080/>), which allows the operating system to designate memory which is available for the taking should the need arise, but which can contain useful data in the mean time. 

This is clearly an area that needs further work. The whole point of virtualization is to isolate guests from each other, but a more cooperative approach to memory requires that these guests, somehow, be aware of the level of contention for resources like memory and respond accordingly. Like high memory and transparent huge pages, balloon drivers may eventually be consigned to the pile of failed technologies. Until something better comes along, though, we'll still need them.
