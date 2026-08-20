---
title: "Justifying FS-Cache"
url: https://lwn.net/Articles/312708/
date: "December 24, 2008"
category: "FS-Cache; Network filesystems-Caching layer"
author: "By Jake Edge December 24, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
December 24, 2008

In what must seem like a never-ending effort, David Howells is once again trying to get a generic mechanism to do local caching for network filesystems into the kernel. The latest version, number 41, of his [FS-Cache patches](<http://lwn.net/Articles/308113/>) was posted back in November, so now he is [asking for it to be added to linux-next](<http://lwn.net/Articles/312227/>). That would mean that the feature was on-track for the mainline in 2.6.29, but it would appear that 2.6.30--if ever--is more likely. 

The idea behind FS-Cache is to create a way for "slow" filesystems to cache their data on the local disk, so that repeated accesses do not require accessing the underlying slow storage. Howells has been working on getting it into the kernel for a number of years; our [first article](<http://lwn.net/Articles/100321/>) about it appeared in 2004. The canonical example of where it might be useful is a network filesystem on a heavily-used or low bandwidth link--the cost of re-reading data from the network may be much higher than retrieving it from a local disk. In addition, the cache can be persistent across reboots, allowing some files to live locally for a very long time. 

But, Howells already has a fairly large, intrusive patch that is headed for 2.6.29: [credentials](<http://lwn.net/Articles/287091/>). That patch touches a lot of code in the kernel, in particular the VFS layer. Christoph Hellwig is [concerned](<https://lwn.net/Articles/312736/>) about both credentials and FS-Cache going in at the same time : 

I don't think we want fscache for .29 yet. I'd rather let the credential code settle for one release, and have more time for actually reviewing it properly and have it 100% ready for .30. 

While that would delay the addition of FS-Cache, Andrew Morton has a [larger concern](<https://lwn.net/Articles/312737/>): 

I don't believe that it has yet been convincingly demonstrated that we want to merge it at all. 

It's a huuuuuuuuge lump of new code, so it really needs to provide decent value. Can we revisit this? Yet again? What do we get from all this? 

Morton is worried about adding additional maintenance headaches with no--or limited--benefits. Using a local disk to cache data from a remote disk is only useful in some scenarios; it can certainly make things worse in others. As Howells [puts it](<https://lwn.net/Articles/312738/>): ""It's a compromise: a trade-off between the loading and latencies of your network vs the loading and latencies of your disk; you sacrifice disk space to make up for the deficiencies of your network."" What Morton is [looking for](<https://lwn.net/Articles/312739/>) is a push from users, be that end users or distributions that are shipping the feature. He would also like to see some benchmarks that show what gain there is when using FS-Cache. 

Howells has patiently answered these concerns, pointing at [some benchmarks](<http://lwn.net/Articles/312722/>) he had posted in November that showed some significant savings. The benchmarks used NFS over a deliberately slow link (to simulate a heavily used network) and showed a huge decrease in the time required to read a large file, but was essentially break-even when operating on a kernel tree. In the kernel tree benchmark, though, the reduction in network traffic was significant. 

More importantly, perhaps, is the fact that Red Hat has shipped FS-Cache in RHEL 5 and there are customers using it, as well as customers interested in using it as Howells [pointed out](<https://lwn.net/Articles/312738/>): 

We (Red Hat) have shipped it in RHEL-5 and some Fedora releases. Doing so is quite an effort, though, precisely because the code is not yet upstream. We have customers using it and are gaining more customers who want it. There even appear to be CentOS users using it (or at least complaining when it breaks). 

While shipping out-of-tree code is no guarantee that the feature will get merged--AppArmor is an excellent counterexample--actual users whose needs are being met by a particular feature _are_ a fairly persuasive argument. Howells [outlines some customer use cases](<https://lwn.net/Articles/312764/>) for FS-Cache, for example: 

We have a number of customers in the entertainment industry who use or would like to use this caching infrastructure in their render farms. They use NFS to distribute textures (say a million and a quarter files) to the individual rendering units. FS-Cache allows them to reduce the network load by satisfying subsequent NFS READ requests from each rendering unit's local cache rather than having to go to the network again. 

In all, it would seem that Morton's concerns were addressed. Whether that means the path is clear for 2.6.30 or these or other concerns will come to the fore is a question that will likely have to wait another three months or so.
