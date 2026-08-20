---
title: Batch processing of network packets
url: https://lwn.net/Articles/763056/
date: "August 21, 2018"
category: "Networking-Performance"
author: "By Jonathan Corbet August 21, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 21, 2018

It has been understood for years that kernel performance can be improved by doing things in batches. Whether the task is freeing memory pages, initializing data structures, or performing I/O, things go faster if the work is done on many objects at once; many kernel subsystems have been reworked to take advantage of the efficiency of batching. It turns out, though, that there was a piece of relatively low-hanging fruit at the core of the kernel's network stack. The 4.19 kernel will feature some work increasing the batching of packet processing, resulting in some impressive performance improvements. 

Once upon a time, network interfaces would interrupt the processor every time a packet was received. That may have worked well with the kind of network interfaces we had in the 1990s, but an interface that worked that way now would be generating many thousands of interrupts per second. That, in turn, would swamp the CPU and prevent any work from getting done. The response to this problem in network circles was [the adoption of an API called "NAPI"](<https://lwn.net/Articles/30107/>) (for "new API") during the long 2.5 development series. 

Old-timers on the net — like your editor — used to have their computers beep at them every time an email arrived. Most of us stopped doing that long ago; the beeps were nonstop, and things reached a point where we simply knew there would be email waiting anytime we got over our dread and opened a mail client. NAPI follows a similar approach: rather than poke the processor when packets arrive, the interface just lets them accumulate. The kernel will then poll the interface occasionally, secure in the knowledge that there will always be packets waiting to be processed. Those packets are then processed in a batch, with the batch size limited by the "weight" assigned to the interface. 

At this level, we can see that batching of packet processing was added some fifteen years ago. But that is where the batching stops; when the NAPI poll happens, the device driver will pass each packet into the network stack with a call to [`netif_receive_skb()`](<https://elixir.bootlin.com/linux/latest/source/net/core/dev.c#L4771>). From that point on, each packet is handled independently, with no further batching. In retrospect, with all of the effort that has gone into streamlining packet processing, one might wonder why that old API was never revisited, but that is how things often go in the real world. 

Eventually, though, somebody usually notices an issue like that; in this case, that somebody was Edward Cree, who put together [a patch set](<https://lwn.net/ml/netdev/5bf84d99-8f77-54ee-7543-ada13a730361@solarflare.com/>) changing how low-level packet reception works. The first step was to supplement `netif_receive_skb()` with a batched version that reads, in its entirety: 

```
void netif_receive_skb_list(struct list_head *head)
        {
    	struct sk_buff *skb, *next;
    
    	list_for_each_entry_safe(skb, next, head, list)
    		netif_receive_skb(skb);
        }
```

Now, rather than calling `netif_receive_skb()` for every incoming packet, a driver can make a list out of a batch of packets and pass them upward with a single call. Not much has changed at this point, but even this tweak improves performance by quite a bit, as it turns out. 

The rest of the patch series is occupied with pushing the batching further up the network stack, so that packets can be passed in lists as far as possible. That gets a little trickier at the higher levels, since some packets have to be handled in fundamentally different ways. For example, some may have been allocated from the system's memory reserves (part of a mechanism to [avoid deadlocks](<https://lwn.net/Articles/195416/>) on network block devices); those require special handling. When such situations are encountered, the list of packets must be split into smaller lists, but the batching is preserved as far as possible. 

The benchmark results (included in [this merge commit](<https://git.kernel.org/linus/2d1b138505dc29bbd7ac5f82f5a10635ff48bddb>)) are interesting. In one test case, using a single receive queue, a kernel with these patches (and a suitably patched driver) showed a 4% improvement in packet-processing speed. That would certainly justify the addition of this bit of infrastructure, but it turns out that this number is the worst case that Cree could find. In general, just adding and using `netif_receive_skb_list()` improves performance by 10%, and the performance improvement with the entire patch series centers around 25%. One test showed a 35% speed improvement. In an era where developers have sweated mightily for much smaller gains, this is an impressive performance improvement. 

One might well wonder why even the simplest batching shown above can improve things by so much. It mostly comes down to cache behavior. As Cree notes in the patch introduction, the processor's instruction cache is not large enough to hold the entire packet-processing pipeline. A device driver will warm the cache with its own code, but then the processing of a single packet pushes that code out of cache, and the driver must start cold with the next one. Just eliminating that bit of cache contention by putting the packets into a list before handing them to the network stack thus improves things considerably; creating the same sort of cache efficiency through the network stack improves things even more. 

Networking also uses a lot of indirect function calls. These calls were never cheap, but the addition of [retpolines](<https://lwn.net/Articles/743265/>) for Spectre mitigation has made things worse. Batching replaces a bunch of per-packet indirect calls with single per-list calls, reducing that overhead. 

There is a problem that often comes with throughput-oriented optimizations, and which can often be seen with batching: an increase in latencies. In the networking case, though, that cost was already paid years ago when NAPI was added. The new batching works on bunches of packets that have already been accumulated at the NAPI poll time and doesn't really add any further delays. So it's an almost free improvement from that point of view. 

This code has been merged for the 4.19 kernel, so it will be generally available when the release happens. As of this writing, only the Solarflare network interfaces use the new `netif_receive_skb_list()` API. The necessary changes at the driver level are quite small, though, so it would be surprising if other drivers were not updated in the relatively near future, possibly even before the 4.19 release. This particular fruit is hanging too low to go unpicked for long.
