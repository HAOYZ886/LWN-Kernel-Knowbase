---
title: A way forward for BFQ
url: https://lwn.net/Articles/709202/
date: "December 14, 2016"
category: "Block layer-IO scheduling"
author: "By Jonathan Corbet December 14, 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 14, 2016

The [BFQ ("budget fair queuing") I/O scheduler](<https://lwn.net/Articles/674308/>) has seen several discussions on the kernel mailing lists since its debut in 2014. By changing the scheduling algorithm and adding a number of heuristics, BFQ is able to provide much more responsive block I/O behavior, especially when dealing with rotational devices. Despite its merits, though, BFQ has never found its way into the mainline. Now, however, it looks like the primary roadblock is about to be removed. 

The objections to BFQ have varied over the years. Initially it was proposed as a new scheduler to be added to the four that the kernel already supports; at that time, BFQ developer Paolo Valente was told that the BFQ functionality should be added to the existing CFQ scheduler instead. In February 2016, he came back with a patch set that attempted to do that, but those patches were (necessarily) intrusive and did not get far. Among other things, reviewers [complained](<https://lwn.net/Articles/709204/>) about replacing the well-tested CFQ scheduler. The bigger complaint, though, which was echoed after [the most recent posting](<https://lwn.net/Articles/704648/>) in October, is that BFQ uses the legacy kernel block API rather than the newer [multiqueue API](<https://lwn.net/Articles/552904/>). 

The multiqueue interface was added in response to scalability issues in the block layer; it allows I/O request queues to be split into multiple subqueues that can be handled on a per-CPU basis. It is seen as the way forward, and it is natural for the block-layer developers to want to see a large piece of new infrastructure using it. There is only one little difficulty, though: BFQ is an I/O scheduler, and the multiqueue code has no provision for I/O scheduling. Leaving out scheduling might make sense if the multiqueue API were to be used only with high-end, solid-state drives, but it also makes that API mostly useless for rotational devices. And, unsurprisingly, it makes the integration of a new I/O scheduler hard. 

If rotational storage were on its last legs, neglecting to properly support it in the future block API might make sense. But, despite the ongoing advances in solid-state storage devices, rotational storage seems likely to stay around for some time yet. That suggests that adding I/O scheduling to the multiqueue API would be a good thing to do. 

Making such an addition was suggested to Valente, but he [did not feel qualified](<https://lwn.net/Articles/709207/>) to do the job. A developer who _is_ qualified is Jens Axboe, the maintainer of the block layer and the author of the multiqueue API. So his recent posting of [multiqueue I/O scheduling support](<https://lwn.net/Articles/708465/>) is a good sign for anybody who is interested in seeing the BFQ scheduler find its way into the mainline. This work is still in an early stage, but Valente [is already working](<https://lwn.net/Articles/709209/>) on porting BFQ over to this new API; Axboe is offering ongoing help in getting that work done. 

So the final chapter in this long story appears to be coming into focus. BFQ will almost certainly find its way in as the first nontrivial I/O scheduler for the multiqueue block API. It is not clear whether there will ever be a need for any others. CFQ, which works well with the legacy APIs, will stay in place without big changes, but it will slowly lose relevance over time. It's a reasonably happy ending that might have been happier had it happened a year or two ago, but sometimes these things take time.
