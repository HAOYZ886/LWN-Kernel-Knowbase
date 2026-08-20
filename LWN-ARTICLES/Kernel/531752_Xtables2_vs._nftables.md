---
title: Xtables2 vs. nftables
url: https://lwn.net/Articles/531752/
date: "January 9, 2013"
category: "Networking-Packet filtering; Nftables"
author: "By Jake Edge January 9, 2013"
---

By **Jake Edge**  
January 9, 2013

The Linux kernel's firewall and packet filtering support has seen quite a few changes over the years. Back in 2009, it looked like a new packet filtering mechanism, [nftables](<https://lwn.net/Articles/324989/>), was set to be the next generation solution for Linux. It was [mentioned](<https://lwn.net/Articles/413036/>) at the 2010 Kernel Summit as a solution that might apply more widely than just for netfilter. But nftables development stalled, until it was [resurrected](<https://lwn.net/Articles/531876/>) in 2012 by netfilter maintainer Pablo Neira Ayuso. During the lull, however, another, more incremental change to the existing netfilter code had been developed; [Xtables2 was proposed for merging](<https://lwn.net/Articles/531877/>) by Jan Engelhardt in mid-December. Many of the same problems in the existing code are targeted by both solutions, so it seems likely that just one or the other gets merged--the decision on which is the subject of some debate. 

Xtables2 has been under development since 2010 or so; Engelhardt gave a [presentation [PDF]](<http://inai.de/documents/Love_for_blobs.pdf>) on it at the 2010 Netfilter workshop. Over the last few years, he has occasionally posted progress reports, but the pace of those (and development itself) seems to have picked up after Neira posted his intention to restart nftables development back in October. Given that it will be difficult--impossible, really--to sell two new packet filtering schemes, either the two will need to come together somehow, or one will have to win out. 

At least so far, Neira and Engelhardt don't agree about the direction that netfilter should take. After the October nftables announcement, Engelhardt [pointed out](<https://lwn.net/Articles/531969/>) that one of the missing nftables features noted by Neira was already available in Xtables2: ""atomic table replace and atomic dump"". Neira's [suggestion](<https://lwn.net/Articles/531968/>) that Engelhardt look at adding the feature to nftables was rebuffed. Beyond that, though, Neira also [said](<https://lwn.net/Articles/531972/>) that it would be ""*extremely hard* to justify its [Xtables2's] inclusion into mainline"". To Engelhardt, and perhaps others, that sounded like a pre-judgment against merging Xtables2, which ""would be really sad"", he [said](<https://lwn.net/Articles/531973/>). He continued by listing a number of useful features already available in Xtables2, including network namespace support, a netlink interface, read-copy update (RCU) support, atomic chain and table replacement, and more. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Much of Neira's announcement concerned the "compatibility layer" that will be needed for any replacement of the existing netfilter code. There are far too many users of `iptables` to leave behind--not to mention the "no ABI breakage" kernel rule. So, for some period of time, both the existing code that supports `iptables` and any new solution will have to coexist in the kernel. Eventually, the older code can be removed. 

One the main problems that both nftables and Xtables2 address is the code duplication that exists in the existing netfilter implementation (which is often referred to as "xtables"). Because that code is protocol-aware, it was essentially necessary to make four copies of much of it in the kernel to handle each different use case: IPv4, IPv6, ARP, and ethernet bridging. That is clearly sub-optimal, and something that both Xtables2 and nftables address. The difference is in _how_ they address it. With Xtables2, a single protocol-independent table is used per network namespace, while nftables defines a new virtual machine to process packets. Essentially, Xtables2 (as its name would imply) is an evolution of the existing code, while nftables is a more fundamental rework of Linux packet filtering. 

That difference in approaches is evident in the discussion over Engelhardt's merge request. Neira is [not impressed with the feature set](<https://lwn.net/Articles/532077/>), but he also [complains](<https://lwn.net/Articles/532078/>) that Xtables2 ""inherits many of the design decisions that were taken while designing iptables back in the late nineties"". As might be guessed, Engelhardt [saw things differently](<https://lwn.net/Articles/532084/>): 

nf_tables itself retains some "late nineties" design decisions as well. 

In my opinion, there is nothing wrong with keeping some concepts. A developer is not required to reevaluate and reinnovate every concept there has been just for the heck of it. (The old "evolution, not revolution" credo.) Throwing everything overboard generally does not turn out to work these days. 

Nftables is hardly a revolution, Neira [said](<https://lwn.net/Articles/532139/>), because it implements backward compatibility: ""revolutions are never backward compatible"". Further discussion noted a number of conceptual similarities between the two approaches, with each side predictably arguing that their solution could do most or all of what the other could do. 

There are some differences between the two, though. For one thing, Xtables2 seems further along, both with its code and with things like [documentation](<http://inai.de/nfxt/Xtables2_Userdoc.txt>) and a [roadmap](<http://inai.de/nfxt/roadmap.php>). As Neira [noted](<https://lwn.net/Articles/532140/>), the development hiatus for nftables clearly set the project back a ways, but he is not ready to provide more details quite yet: 

I understand you want to know more on the future of nftables, but because the way I am, I prefer to skip "hot air" wording by now and talk on code done anytime soon. 

So I have to request some patience from you. We promise to deliver as much information as possible once we are done with the core features. 

So there are two competing proposals for how to move forward with netfilter, one that is largely ready (though certainly not complete), according to Engelhardt, and one that is still under active "core" development. While once seen as the likely successor, nftables certainly suffered from lack of attention for a few years, while Xtables2 was seemingly chugging along for much of that time. 

Clearly, both Engelhardt and Neira favor their chosen solutions, and neither seems likely to join forces with the other. Engelhardt [indicated](<https://lwn.net/Articles/532145/>) that he isn't advocating dropping nftables, necessarily, but is instead focused on getting Xtables2 features into the hands of users: 

In all fairness, I have never said anything about dropping nft. I am focused on xt2, its inclusion and subsequent maintenance, because it resolves the ipt shortcomings in a way that I think appeals most to the userspace crowd. 

Neira [proposed](<https://lwn.net/Articles/532146/>) a discussion at this year's Netfilter workshop (which should happen by mid-year) to try to resolve the issue. While Engelhardt expressed some concern over the wait, a few months may be needed as Jozsef Kadlecsik [pointed out](<https://lwn.net/Articles/532147/>): ""Both nftables and xtables2 have got nice features, so it's not a simple question."" Network maintainer David Miller concurred with Neira's proposal. 

While it may be technically possible to merge Xtables2 and start down the path toward removing the older interface, then switch to nftables when it is ready, it seems an unlikely outcome. If the netfilter developers (and maintainers) are convinced that nftables is the right way forward, skipping the Xtables2 step makes sense. That may mean a longer delay before some of the longstanding problems in the existing code are addressed, but trying to maintain three different packet filtering schemes in the kernel is simply not going to happen.
