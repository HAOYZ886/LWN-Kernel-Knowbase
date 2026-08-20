---
title: The Grand Unified Flow Cache
url: https://lwn.net/Articles/194443/
date: "August 7, 2006"
category: "Grand Unified Flow Cache; Networking-Channels"
author: 
---

The Grand Unified Flow Cache is one of those items which shows up as a bullet in networking summit presentations; the networking folks appear to know what it means, but they have been somewhat remiss in documenting the idea for the rest of us. This concept has returned in the context of the network channels discussion, and enough hints have been dropped to let your editor - who is not afraid to extrapolate a long way from minimal data - get a sense for what the term means. Should it be implemented, the GUFC could bring significant changes to the entire networking stack. 

The net channel concept requires that the kernel be able to quickly identify the destination of each packet and drop it into the proper channel. Even better would be to have a smart network adapter perform that classification as the packet arrives, taking the kernel out of that part of the loop altogether. One way of performing this classification would be to form a tuple from each packet and use that tuple as a lookup key in some sort of fast data structure. When a packet's tuple is found in this structure (the flow cache), its fate has been determined and it can be quickly shunted off to where it needs to be. 

This tuple, as [described](<https://lwn.net/Articles/194461/>) by Rusty Russell, would be made up of seven parameters: 

  * The source IP address 
  * The destination IP address 
  * A bit indicating whether the source is local 
  * A bit indicating whether the destination is local 
  * The IP protocol number 
  * The source port 
  * The destination port 

These numbers, all together, are sufficient to identify the connection to which any packet belongs. A quick lookup on an incoming packet should, thus, yield a useful destination (such as a network channel) for that packet with no further processing. 

Features like netfilter mess up this pretty picture, however. Within the kernel, netfilter is set up such that every packet is fed to the appropriate chain(s). As soon as every packet has to go through a common set of hooks, the advantage of the GUFC is lost. Rusty's description of the problem is this: 

The mistake (?) with netfilter was that we are completely general: you will see all packets, do what you want. If, instead, we had forced all rules to be of form "show me all packets matching this tuple" we would be in a [position to] combine it in a single lookup with routing etc. 

So, the way around this problem would be to change the netfilter API to work better with a grand unified flow cache. Rules could be written in terms of the above tuples (with wild cards allowed), and only packets which match the tuples need pass through the (slow) netfilter path. That would allow packets which are not of interest to the filtering code to bypass the whole mechanism - and the decision could be made in a single lookup. 

Often, however, a packet filtering decision can be made on the basis of the tuple itself - once a packet matches the tuple, there is no real need to evaluate it against the rule separately. So, for example, once the connection tracking code has allowed a new connection to be established, and a tuple describing that connection has been added to the cache, further filtering for that connection should not be required. If netfilter and the flow cache worked together effectively, the per-packet overhead could be avoided in many cases. 

One way this might work would be to have a set of callbacks invoked for each tuple which is added to the flow cache. A module like netfilter could examine the tuple relative to the current rule set and let the kernel know if it needs to see packets matching that tuple or not. Then, packets could be directed to the appropriate filters without the need for wildcard matching in the tuple cache. 

There is a small cost to all of this: 

Of course, it means rewriting all the userspace tools, documentation, and creating a complete new infrastructure for connection tracking and NAT, but if that's what's required, then so be it. 

Rusty has never let this sort of obstacle stop him before, so all of this might just happen. 

But probably not anytime soon. There's a long list of questions which need to be answered before a serious implementation attempt is made. Whether it would truly perform as well as people hope is one of them; these schemes can get quite a bit slower once all of the real-world details are factored in. Rule updates could be a challenge; an administrator who has just changed packet filtering rules is unlikely to wait patiently while the new rules slowly work their way into the cache. Finding a way to get the hardware to help in the classification process will not be entirely straightforward. And so on. But it would seem that there are a number of interesting ideas in this area. That is bound to lead to good stuff sooner or later.
