---
title: Bulk network packet transmission
url: https://lwn.net/Articles/615238/
date: "October 8, 2014"
category: "Networking-Performance"
author: "By Jonathan Corbet October 8, 2014"
---

By **Jonathan Corbet**  
October 8, 2014

One of the keys to good performance on contemporary systems is batching — getting a lot of work done relative to a given fixed cost. If, for example, a lock must be acquired to do one unit of work in a specific subsystem, doing multiple units of work while the lock is held will reduce the overall overhead of the system. Much of the scalability work that has been done in recent years has, in some way, related to increasing batching where possible. Some recent changes in the networking subsystem show that batching can improve performance there as well. 

Every time a packet is transmitted over the network, a sequence of operations must be performed. These include acquiring the lock for the queue of outgoing packets, passing a packet to the driver, putting the packet in the device's transmit queue, and telling the device to start transmitting. Some of those operations are inherently per-packet, but others are not. The acquisition of the queue lock could be amortized across multiple packet transmissions, for example, and the act of telling the device to start transmission may be expensive indeed. It can involve hardware operations or, even, on some systems, hypervisor calls. 

Often, when there is one packet to transmit, there are others waiting in the queue as well; network traffic can be inherently bursty. So it would make sense for the networking stack to try to split the fixed costs of starting packet transmission across as many packets as possible. Some techniques, such as segmentation offload (wherein the network interface splits large chunks of data into packets) perform that kind of batching. But, in current kernels, if the networking stack has a set of packets ready to go, they will be sent out the slow way, one at a time. 

That situation will begin to change in 3.18, when a relatively small set of changes will be merged. Consider the function exported by drivers now to send a packet: 

```
netdev_tx_t	(*ndo_start_xmit) (struct sk_buff *skb, struct net_device *dev);
```

This function takes the packet pointed to by `skb` and transmits it via the specified `dev`. Every call is a standalone operation, with all the associated fixed costs. The initial plan for 3.18 was to specify a new function that drivers could provide: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

```
void (*ndo_xmit_flush)(struct net_device *dev, u16 queue);
```

If a driver provided this function, it was indicating to the networking stack that it is prepared for (and can benefit from) batched transmission. In this case, the networking stack could make multiple calls to `ndo_start_xmit()` to queue packets for transmission; the driver would accept them, but not actually start the transmission operation. At the end of a sequence of such calls, `ndo_xmit_flush()` would be called to indicate the end; at that point, actual hardware transmission would be started. 

There were concerns, though, that putting another indirect function call into the transmit path would add too much overhead, so this particular function was ripped out almost as soon as it landed in the net-next repository. In its place, the `sk_buff` structure has gained a new Boolean variable called `xmit_more`. If that variable is true, then there are more packets coming and the driver can defer starting hardware transmission. This variable takes out the extra function call while making the needed information available to drivers that can make use of it. 

This mechanism, added by David Miller, makes batching possible. A couple of drivers were fixed to support batching, but David did not change the networking stack to actually do the batching. That work fell to Jesper Dangaard Brouer, whose [bulk dequeue support patches](<https://lwn.net/Articles/615240/>) have also been merged for 3.18. This work, too, is limited in scope; in particular, it will only work with queuing disciplines that have a single transmit queue. 

The change Jesper made is simple enough: in a situation where a packet is being transmitted, the stack will attempt to send out a series of packets together while the queue lock is held. The [byte queue limits](<https://lwn.net/Articles/454390/>) mechanism is used to put an upper bound on the amount of data that can be in flight at once. Once the limit is hit (or the supply of packets runs out), `skb->xmit_more` will be set to false and the traffic will be on its way. 

Eric Dumazet looked at the patch set and realized that things could be taken a bit further: the process of validating packets for transmission could be moved outside of the queue lock entirely, increasing concurrency in the system. The resulting [patch](<https://lwn.net/Articles/615243/>) had benefits that Eric described as [awesome](<https://lwn.net/Articles/615242/>): full 40Gb/sec wire speed, even in the absence of segmentation offload. Needless to say, this patch, too, has been accepted into the net-next tree for the 3.18 merge window. 

All told, the changes are relatively small. But small changes can have big effects when they are applied to the right places. These little changes should help to ensure that the networking stack in the 3.18 release is the fastest yet.
