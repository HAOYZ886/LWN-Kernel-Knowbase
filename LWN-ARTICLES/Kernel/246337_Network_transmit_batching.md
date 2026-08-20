---
title: Network transmit batching
url: https://lwn.net/Articles/246337/
date: "August 22, 2007"
category: "Device drivers-Network drivers; Networking"
author: "By Jonathan Corbet August 22, 2007"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 22, 2007

At the core of most network drivers is the `hard_start_xmit()` method, which is called once for every packet which is to be transmitted. This method will normally acquire locks and insert the packet into the adapter's transmit queue. As a rule, outgoing packets do not accumulate in the kernel; they are handed to the driver, one at a time, when they are ready to go. There are times, though, when packets cannot be handed off immediately. If, for example, the hardware transmit queue is currently full, the networking subsystem will have to hold on to the packet until things clear out. Once the driver is able to accept packets for the device again, the one-at-a-time behavior will resume. 

The networking developers are always looking for ways to squeeze a little more performance from their code. Krishna Kumar took a look at the behavior described above and wondered: why not pass the list of accumulated packets to the driver in a single call? Batching of transmission operations in this way has the potential to minimize the cost of locking and device preparation overhead, making packet transmission as a whole more efficient. To explore this idea, Krishna has posted a few versions of the [SKB batching patch set](<http://lwn.net/Articles/246333/>). 

Implementing SKB batching requires a couple of driver API changes - but they are small and only required for batching-aware drivers. The first step is to set the `NETIF_F_BATCH_SKBS` bit in the `features` field of the `net_device` structure. That flag tells the network stack that the driver can handle batched transmissions. 

The prototype for `hard_start_xmit()` is: 

```
int (*hard_start_xmit)(struct sk_buff *skb, struct net_device *dev);
```

That prototype does not change, but a driver which has indicated that batching is acceptable for `dev` may find its `hard_start_xmit()` method called with `skb` set to `NULL`. The `NULL` value is an indication that there is a batch of packets to transmit; that batch will be found enqueued on the (new) list found at `dev->skb_blist`. So the (much simplified) form of a batching-aware driver's `hard_start_xmit()` function will look something like: 

```
driver_specific_locking_and_setup();
        if (skb)
    	ret = send_a_packet(internal_dev, skb);
        else {
    	while ((skb = __skb_dequeue(dev->skb_blist)) != NULL) {
    	    ret = send_a_packet(internal_dev, skb);
    	    if (ret)
    	        break;
            }
        }
        driver_specific_cleanup();
```

The reality of the situation can be a bit more complicated, especially if the driver implements optimizations like suppressing completion interrupts until the last packet of the batch has been sent. But the core of the change is as described here - not a whole lot to it. 

As of this writing, the networking developers are still trying to determine what the performance effects of this patch are. There is particular interest in seeing how batching compares with TCP segmentation offloading, which is also, at its core, a transmission batching mechanism. The proof is very much in the benchmarks for a patch like this; if the results are good enough, the patch will likely be merged.
