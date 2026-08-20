---
title: "LCA: IP address exhaustion and the end of the open net"
url: https://lwn.net/Articles/424696/
date: "January 26, 2011"
category: "Networking-IPv6"
author: "By Jonathan Corbet January 26, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 26, 2011

Geoff Huston is the Chief Scientist at the Asia Pacific Network Information Centre. His frank linux.conf.au 2011 keynote took a rather different tack than [Vint Cerf's talk](<https://lwn.net/Articles/424467/>) did the day before. According to Geoff, Vint is "a professional optimist." Geoff was not even slightly optimistic; he sees a difficult period coming for the net; unless things happen impossibly quickly, the open net that we often take for granted may be gone forevermore. 

The net, Geoff said, is based on two "accidental technologies": Unix and packet switching. Both were new at their time, and both benefited from open-source reference implementations. That openness created a network which was accessible, neutral, extensible, and commercially exploitable. [![\[Geoff Huston\]](https://static.lwn.net/images/conf/2011/lca/GeoffHuston1-sm.jpg)](<https://lwn.net/Articles/424703/>) As a result, proprietary protocols and systems died, and we now have a "networking monoculture" where TCP/IP dominates everything. Openness was the key: IPv4 was as mediocre as any other networking technology at that time. It won not through technical superiority, but because it was open. 

But staying open can be a real problem. According to Geoff, we're about to see "another fight of titans" over the future of the net; it's not at all clear that we'll still have an open net five years from now. Useful technologies are not static, they change in response to the world around them. Uses of technologies change: nobody expected all the mobile networking users that we now have; otherwise we wouldn't be in the situation we're in where, among other things, "TCP over wireless is crap." 

There are many challenges coming. Network neutrality will be a big fight, especially in the US. We're seeing more next-generation networks based around proprietary technologies. Mobile services tend to be based on patent-encumbered, closed applications. Attempts to bundle multiple types of services - phone, television, Internet, etc. - are pushing providers toward closed models. 

#### The real problem

But the biggest single challenge by far is the simple fact that we are out of IP addresses. There were 190 million IP addresses allocated in 2009, and 249 million allocated in 2010. There are very few addresses left at this time: IANA will run out of IPv4 addresses in early February, and the regional authorities will start running out in July. The game is over. Without open addressing, we don't have an open network that anybody can join. That, he said, is "a bit of a bummer." 

This problem was foreseen back in 1990; in response, a nice plan - IPv6 - was developed to ensure that we would never run out of network addresses. That plan assumed that the transition to IPv6 would be well underway by the time that IPv4 addresses were exhausted. Now that we're at that point, how is that plan going? Badly: currently 0.3% of the systems on the net are running IPv6. So, Geoff said, we're now in a position where we have to do a full transition to IPv6 in about seven months - is that feasible? 

To make that transition, we'll have to do more than assign IPv6 addresses to systems. This technology will have to be deployed across something like 1.8 billion people, hundreds of millions of routers, and more. There's lots of fun system administration work to be done; think about all of the firewall configuration scripts which need to be rewritten. Geoff's question to the audience was clear: "you've got 200 days to get this done - what are you doing here??" 

Even if the transition can be done in time, there's another little problem: the user experience of IPv6 is poor. It's slow, and often unreliable. Are we really going to go through 200 days of panic to get to a situation which is, from a user point of view, worse than what we have now? Geoff concludes that IPv6 is simply not the answer in that time frame - that transition is not going to happen. So what will we do instead? 

One commonly-suggested approach is to make much heavier use of network address translation (NAT) in routers. A network sitting behind a NAT router does not have globally-visible addresses; hiding large parts of the net behind multiple layers of NAT can thus reduce the pressure on the address space. But it's not quite that simple. 

[![\[Geoff Huston\]](https://static.lwn.net/images/conf/2011/lca/GeoffHuston2-sm.jpg)](<https://lwn.net/Articles/424705/>) Currently, NAT routers are an externalized cost for Internet service providers; they are run by customers and ISP's need not worry about them. Adding more layers of NAT will force ISPs to install those routers. And, Geoff said, we're not talking about little NAT routers - they have to be really big NAT routers which cannot fail. They will not be cheap. Even then, there are problems: multiple levels of NAT will break applications which have been carefully crafted to work around a single NAT router. How NAT routers will play together is unclear - the IETF refused to standardize NAT, so every NAT implementation is creative in its own way. 

It gets worse: adding more layers of NAT will break the net in fundamental ways. Every connection through a NAT router requires a port on that router; a single web browser can open several connections in an attempt to speed page loading. A large NAT router will have to handle large numbers of connections simultaneously, to the point that it will run out of port numbers. Ports numbers are only 16 bits, after all. So ISPs are going to have to think about how many ports they will make available to each customer; that number will converge toward "one" as the pressure grows. Our aperture to the net, Geoff said, is shrinking. 

So perhaps we're back to IPv6, somehow. But there is no compatibility between IPv4 and IPv6, so systems will have to run both protocols during the transition. The transition plan, after all, assumed that it would be completed before IPv4 addresses ran out. But that plan did not work; it was, Geoff said, "economic rubbish." But we're going to have to live with the consequences, which include running dual stacks for a transition period that, he thinks, could easily take ten years. 

During that time, we're going to have to figure out how to make our existing IPv4 addresses last longer. Those addresses, he said, are going to become quite a bit more expensive. There will be much more use of NAT, and, perhaps, better use of current private addresses. Rationing policies will be put into place, and governmental regulation may well come into play. And, meanwhile, we know very little about the future we're heading into. TCP/IP is a monoculture, there is nothing to replace it. We don't know how long the transition will take, we don't know who the winners and losers will be, and we don't know the cost. We live in interesting times. 

#### An end to openness

Geoff worried that, in the end, we may never get to the point where we have a new, IPv6 network with the same degree of openness we have now. Instead, we may be heading toward a world where we have privatized large parts of the address space. The problem is this: the companies which have lost the most as the result of the explosion of the Internet - the carriers - are now the companies which are expected to fund and implement the transition [![\[Geoff Huston\]](https://static.lwn.net/images/conf/2011/lca/GeoffHuston3-sm.jpg)](<https://lwn.net/Articles/424706/>) to IPv6. They are the ones who have to make the investment to bring this future around; will they really spend their money to make their future worse? These companies have no motivation to create a new, open network. 

So what about the companies which _have_ benefited from the open net: companies like Google, Amazon, and eBay? They are not going to rescue us either for one simple reason: they are now incumbents. They have no incentive to spend money which will serve mainly to enable more competitors. They are in a position where they can pay whatever it takes to get the address space they need; a high cost to be on the net, is, for them, a welcome barrier to entry that will keep competition down. We should not expect help from that direction. 

So perhaps it is the consumers who have to pay for this transition. But Geoff did not see that as being realistic either. Who is going to pay $20/month more for a dual-stack network which works worse than the one they have now? If one ISP attempts to impose such a charge, customers will flee to a competitor which does not. Consumers will not fund the change. 

So the future looks dark. The longer we toy with disaster, Geoff said, the more likely it is that the real loser will be openness. It is not at all obvious that we'll continue to have an open net in the future. He doesn't like that scenario; it is, he said, the worst possible outcome. We all have to get out there, get moving, and fix this problem. 

One of the questions asked was: what can we do? His answer was that we really need to make better software: "IPv6 sucks" currently. Whenever IPv6 is used, performance goes down the drain. Nobody has yet done the work to make the implementations truly robust and fast. As a result, even systems which are capable of speaking both protocols will try to use IPv4 first; otherwise, the user experience is terrible. Until we fix that problem, it's hard to see how the transition can go ahead.
