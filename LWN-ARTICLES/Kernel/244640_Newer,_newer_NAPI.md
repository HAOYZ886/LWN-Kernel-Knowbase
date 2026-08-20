---
title: "Newer, newer NAPI"
url: https://lwn.net/Articles/244640/
date: "August 7, 2007"
category: "NAPI; Networking-NAPI"
author: "By Jonathan Corbet August 7, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 7, 2007

Last December, LWN [looked at a proposal to rework the NAPI interface](<http://lwn.net/Articles/214457/>) used for packet reception in high-bandwidth network drivers. Since then, the interface has gone through some changes, but now appears to be in something close to its final form. Anybody who maintains a NAPI-capable network driver will need to adapt to the new API; in many cases the changes will be simple, but New-NAPI offers some added features which may be of value to drivers of complicated hardware. 

The core idea behind the NAPI interface is that, on a busy network, the kernel does not need to be interrupted every time a network packet arrives. Instead, the kernel can simply poll occasionally in the sure knowledge that packets will be there waiting. Your editor likes to compare packet receive interrupts with the beeps we all had, once upon a time, to let us know when email had arrived. Few of us use those beeps anymore; we have no doubt that there will be email waiting whenever we see fit to look for it. Like us, the kernel can do without unneeded distractions; that is especially true when those distractions can take the form of thousands of interrupts every second. 

There are other advantages to the NAPI approach. If the networking subsystem is overwhelmed and must drop packets, NAPI makes it possible for them to be dropped before they are ever fed into the stack. For various reasons, packet reordering tends to be less of a problem with NAPI as well. 

The new `napi_struct` patch set (currently at [version 5](<http://lwn.net/Articles/244453/>)), like its predecessor, introduces a new structure for controlling packet reception: 

```
struct napi_struct {
    	struct list_head	poll_list;
    	unsigned long		state;
    	int			weight;
    	int			quota;
    	int			(*poll)(struct napi_struct *, int);
    	/* Netpoll-related fields omitted */
        }
```

This structure is no longer part of the `net_device` structure, though; instead, drivers are expected to allocate it separately. Usually it will be part of whatever larger structure the driver uses to represent the device internally. One of the main advantages of this approach is that device drivers can, if need be, create more than one `napi_struct` structure for a given device. Contemporary hardware can support multiple receive queues with nifty features like CPU affinity and flow separation; multiple NAPI structures makes it easier to use those queues efficiently. 

Drivers need not fill in the fields of the `napi_struct` structure, though zeroing the whole structure at allocation time can only be a good idea. Instead, each NAPI instance must be registered with the system with: 

```
void netif_napi_add(struct net_device *dev,
                            struct napi_struct *napi,
    			int (*poll)(struct napi_struct *, int),
    			int weight);
```

Here, `dev` is the `net_device` structure associated with the interface, `napi` is the NAPI structure, `poll()` is the polling method to be used with this instance, and `weight` is the relative weight to be given to this interface. Note that `poll()` and `weight` are no longer part of the `net_device` structure. As always, the setting of `weight` is somewhat arbitrary, with most values varying between 16 (for basic Ethernet) and 64 \- though InfiniBand uses 100. There is talk of reworking weights in a future patch, but that is a separate issue. 

There is no `netif_napi_remove()`, as there is currently no need for it. 

The prototype of the `poll()` method has changed somewhat: 

```
int (*poll)(struct napi_struct *napi, int budget);
```

The NAPI structure comes in as `napi`, of course. The `budget` parameter specifies how many packets the driver is allowed to pass into the network stack on this call. There is no need to manage separate quota fields anymore; drivers should simply respect `budget` and return the number of packets which were actually processed. 

Most of the other NAPI-related functions have had the obvious changes made to their prototypes. The two ways of turning on polling are: 

```
void netif_rx_schedule(struct net_device *dev, 
                               struct napi_struct *napi);
        /* ...or... */
        int netif_rx_schedule_prep(struct net_device *dev,
    			       struct napi_struct *napi);
        void __netif_rx_schedule(struct net_device *dev,
    		       	     struct napi_struct *napi);
```

Polling is turned off with: 

```
void netif_rx_complete(struct net_device *dev,
    			   struct napi_struct *napi);
```

Since there can be more than one `napi_struct` structure in existence, each can have polling enabled independently. Drivers are responsible for disabling polling on all outstanding NAPI structures when the interface is shut down (or when its `stop()` method is called). 

The `netif_poll_enable()` and `netif_poll_disable()` functions no longer exist, since polling is no longer tied to the `net_device` structure. Instead, these functions should be used: 

```
void napi_enable(struct napi *napi);
        void napi_disable(struct napi *napi);
```

Networking maintainer David Miller, who has taken on the development of this patch, says: 

I don't anticipate making any more changes, just fixing bugs. So please help me with that so we can finalize this patch. I intend to cut a net-2.6.24 tree and stuff this patch into it by the end of the week. 

So anybody charged with maintaining out-of-tree network drivers should be prepared for a significant API change in the 2.6.24 kernel.
