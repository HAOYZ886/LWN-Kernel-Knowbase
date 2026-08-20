---
title: Slab reduction
url: https://lwn.net/Articles/389400/
date: "May 26, 2010"
category: "Memory management-Slab allocators"
author: "By Jonathan Corbet May 26, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 26, 2010

LWN took [a brief look](<http://lwn.net/Articles/388296/>) at the new SLEB memory allocator last week. Since then, Christoph Lameter has posted [a second revision of the patch](<http://lwn.net/Articles/388911/>), but subsequent discussion suggests that SLEB is not likely to find its way into the mainline. 

The main detractor at the outset was Nick Piggin, author of the unmerged [SLQB](<http://lwn.net/Articles/311502/>) allocator. Nick saw SLEB as a step forward from SLUB, which he [suggests](<https://lwn.net/Articles/389401/>) should never have been merged: 

I don't think SLUB ever proved itself very well. The selling points were some untestable handwaving about how queueing is bad and jitter is bad, ignoring the fact that queues could be shortened and periodic reaping disabled at runtime with SLAB style of allocator. It also has relied heavily on higher order allocations which put great strain on hugepage allocations and page reclaim (witness the big slowdown in low memory conditions when tmpfs was using higher order allocations via SLUB). 

What Nick would like to see at this point is not another in-kernel slab allocator (not even SLQB), but, instead, an effort to improve slab itself, which, he says, is already pretty close to optimal. And, regardless, Linus has [made it clear](<https://lwn.net/Articles/389402/>) that he's not interested in merging more allocators: 

I'm not going to merge YASA that will stay around for years, not improve on anything, and will just mean that there are some bugs that developers don't see because they depend on some subtle interaction with the sl*b allocator. 

Nick's plan is to start by cleaning up the slab allocator to make the code more approachable than it is now. Then, any performance problems can be carefully fixed up, with an emphasis on not causing performance regressions elsewhere. Over time, he says, we should get farther with a single allocator that is used (and tested) by everybody than by ripping out code with a long development history and replacing it with something new and untried.
