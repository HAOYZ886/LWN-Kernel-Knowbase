---
title: Frontswap gets broadsided
url: https://lwn.net/Articles/465317/
date: "November 2, 2011"
category: Transcendent memory
author: "By Jonathan Corbet November 2, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 2, 2011

"Frontswap" is the second half of Dan Magenheimer's [transcendent memory](<https://lwn.net/Articles/454795/>) concept; the first half ("cleancache") was merged for the 3.0 kernel. Given that the job was halfway done, one might be forgiven for thinking that getting frontswap merged would not be a big challenge, despite the fact that, like many memory-management patches, transcendent memory has had a long and somewhat rocky path into the mainline. Dan must have known better, though, as evidenced by his decision to copy your editor on [the frontswap pull request](<https://lwn.net/Articles/465032/>), nicely providing a front-row seat to the 100+ messages that followed. Some version of this patch set may well make it into the mainline eventually, but it now seems quite unlikely to happen in the 3.2 cycle. 

The core idea behind frontswap is to provide a less expensive alternative to pushing a page out to the swap device. That alternative could be one of a number of possibilities: storing the page (possibly compressed) in a memory pool shared between virtual machines, writing it to an SSD-based intermediate device, or adding a reference to a stored page with duplicate contents, for example. Frontswap is not required to accept a page handed to it, but, if it does accept that page, it must be able to reproduce it on demand in the future. The primary use case appears to be balancing memory use between Xen-based virtual machines, but others can be imagined. 

If one were to look at the initial response to the post, it would appear that there was a groundswell of support for these patches; several messages came in calling for their inclusion. Those messages, however, came from other people at Oracle (Dan's employer) or other large companies, though, and their authors are not normally known for their participation in conversations about memory management code, so they may have had something other than the intended effect. It looked a bit like an organized pressure campaign. When the core kernel developers started to respond, the tone of the conversation changed considerably. 

There were a number of complaints raised. The frontswap patches were not going through the -mm tree, and they did not carry acks from any of the recognized memory management developers, so some people started to suspect that Dan was trying to circumvent the normal processes. There is also a fair amount of doubt about the utility of the patches and the way they operate; Christoph Hellwig, for example, [described](<https://lwn.net/Articles/465324/>) frontswap as ""a bunch of really ugly hooks over core code, without a clear definition of how they work or a killer use case."" Various core memory management developers, their attention drawn by the pull request, found a number of things not to like. 

One other complaint raised was the lack of any sort of associated benchmarks. Frontswap is, in the end, a sort of performance-enhancing patch; such changes are normally expected to be accompanied by test results showing that performance is indeed enhanced for the target workloads. Equally important is showing that performance is not hurt on other workloads - always a big concern when making changes to memory management behavior. For this kind of change, it is important to show that there is no impact on systems where the new facility is not used at all; Dan has not yet done that. 

Chances are good that satisfying benchmark results can be produced eventually, and that the technical objections that have been raised can be fixed. Even then, though, frontswap is unlikely to get an immediate green light for merging into the mainline, for a few reasons. One is that life is never easy for those making core memory management changes; experience has shown that it is far too easy to make mistakes that only show up many months later when somebody tries their important workload on a new kernel. Dan has [complained](<https://lwn.net/Articles/465327/>) about the "hazing" he has gone through, but he has had an easier time than [some others](<https://lwn.net/Articles/224884/>). 

That said, Dan's life has not been improved by the association of his work with Xen which, while being free software that is now mostly in the mainline, is still looked upon dimly by many developers. His interaction style also sometimes does not help. Finally, Dan has, by virtue of unfortunate timing that is not at all his fault, run into another problem that was best [explained](<https://lwn.net/Articles/465333/>) by Andrew Morton: 

At kernel summit there was discussion and overall agreement that we've been paying insufficient attention to the big-picture "should we include this feature at all" issues. We resolved to look more intensely and critically at new features with a view to deciding whether their usefulness justified their maintenance burden. It seems that you're our crash-test dummy ;) 

Dan had not explicitly volunteered for that role, but, then, few people (or dummies) ever do. But, at this point, the process will have to play out on those terms. Barring some sort of surprising executive decision by Linus, this particular discussion is unlikely to come to a resolution before the close of the 3.2 merge window. 

Another idea discussed at the kernel summit was that code that is in active use, and that is shipped by distributors over the long haul, should probably find its way into the mainline eventually even if it is not entirely pleasing. Transcendent memory has been in the openSUSE kernel since 2009, and has been shipped by Oracle for some time as well. Clearly, some people see value in this work. Given time, patience, and a willingness to address technical issues, that should be sufficient to get this capability into the mainline eventually.
