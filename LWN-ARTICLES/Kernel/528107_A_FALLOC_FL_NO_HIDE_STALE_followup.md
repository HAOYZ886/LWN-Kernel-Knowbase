---
title: A FALLOC_FL_NO_HIDE_STALE followup
url: https://lwn.net/Articles/528107/
date: "December 5, 2012"
category: "Development model-Code review; fallocate"
author: "By Jonathan Corbet December 5, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 5, 2012

Last week's edition included [an article on the addition of the `FALLOC_FL_NO_HIDE_STALE` flag](<https://lwn.net/Articles/527373/>) to the `fallocate()` system call. Some developers, objecting to the patch and the way it got into the kernel, had called for it to be reverted before the 3.7 release went final. At the time, Linus had not made any remarks in the discussion or indicated whether he would accept the revert. 

That situation changed after Linus was [prompted](<https://lwn.net/Articles/528108/>) by Martin Steigerwald. His [response](<https://lwn.net/Articles/528109/>) was clear enough: 

If you want something reverted, you show me the *technical* reason for it. Not the "ooh, I'm so annoyed by how this was done" reason for it. 

And if your little feelings got hurt, get your mommy to tuck you in, don't email me about it. Because I'm not exactly known for my deep emotional understanding and supportive personality, am I? 

There were some technical reasons offered in the discussion, along with the more general process-oriented complaints. But it seems clear that Linus has not found that discussion convincing. So, in the absence of a surprise from somewhere, it seems that the new `fallocate()` flag will remain for the 3.7 release, at which point it will become part of the kernel's user-space ABI.
