---
title: Overlayfs for 3.10
url: https://lwn.net/Articles/542707/
date: "March 13, 2013"
category: "Filesystems-Union; Overlayfs"
author: "By Jonathan Corbet March 13, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 13, 2013

The "overlayfs" filesystem is one implementation of the [union filesystem](<https://lwn.net/Articles/324291/>) concept, whereby two or more filesystems can be combined into a single, virtual tree. LWN first [reported on](<https://lwn.net/Articles/403012/>) overlayfs in 2010; since then it has seen continued development and has been shipped by a number of distributors. It has not, however, managed to find its way into the mainline kernel. 

In a recent [posting of the overlayfs patch set](<https://lwn.net/Articles/542551/>), developer Miklos Szeredi asked if it could be considered for inclusion in the 3.10 development cycle. He has made such requests before, but, this time, Linus [answered](<https://lwn.net/Articles/542709/>): 

Yes, I think we should just do it. It's in use, it's pretty small, and the other alternatives are worse. Let's just plan on getting this thing done with. 

At Linus's request, Al Viro has [agreed](<https://lwn.net/Articles/542710/>) to review the patches again, though he noted that he has not been entirely happy with them in the past. Unless something serious and unfixable emerges from that review, it looks like overlayfs is finally on track for merging into the mainline kernel.
