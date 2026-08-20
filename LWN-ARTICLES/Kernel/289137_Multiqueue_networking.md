---
title: Multiqueue networking
url: https://lwn.net/Articles/289137/
date: "July 8, 2008"
category: Networking
author: "By Jonathan Corbet July 8, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 8, 2008

One of the fundamental data structures in the networking subsystem is the transmit queue associated with each device. The core networking code will call a driver's `hard_start_xmit()` function to let the driver know that a packet is ready for transmission; it is then the driver's job to feed that packet into the hardware's transmit queue. The result is a data structure which looks vaguely like this: 

> ![\[Network transmit queue\]](https://static.lwn.net/images/ns/kernel/net-tx-old.png)

"Vaguely" because the list of `sk_buff` structures (SKBs - the internal representation of packets) does not exist in this form within the kernel; instead, the driver maintains the queue in a way that the hardware can process it. 

This is a scheme which has worked well for years, but it has run into a fundamental limitation: it does not map well to devices which have multiple transmit queues. Such devices are becoming increasingly common, especially in the wireless networking area. Devices which implement the Wireless Multimedia Extensions, for example, can have four different classes of service: video, voice, best-effort, and background. Video and voice traffic may receive higher priority within the device - it is transmitted first - and the device can also take more of the available air time for such packets. On the other hand, the queues for this kind of traffic may be relatively short; if a video packet doesn't get sent on its way quickly, the receiving end will lose interest and move on. So it might be better to just drop video packets which have been delayed for too long. 

On the other hand, the "background" level only gets transmitted if there is nothing else to do; it is well-suited to low-priority traffic like bittorrent or email from the boss. It would make sense to have a relatively long queue for background packets, though, to be able to take full advantage of a lull in higher-priority traffic. 

Within these devices, each class of service has its own transmit queue. This separation of traffic makes it easy for the hardware to choose which packet to transmit next. It also allows independent limits on the size of each queue; there is no point in filling the device's queue space with background traffic which is not going to be transmitted in any case. But the networking subsystem does not have any built-in support for multiqueue devices. This hardware has been driven using a number of creative techniques which have gotten the job done, but not in an optimal way. That may be about to change, though, with the advent of David Miller's [multiqueue transmit patch series](<http://lwn.net/Articles/288940/>). 

The current code treats a network device as the fundamental unit which is managed by the outgoing packet scheduler. David's patches change that idea somewhat, since each transmit queue will need to be scheduled independently. So there is a new `netdev_queue` structure which encapsulates all of the information about a single transmit queue, and which is protected by its own lock. Multiqueue drivers then set up an array of these structures. So the new data structure can, with sufficient imagination, be seen to look something like this: 

> ![\[Multiqueue tx structure\]](https://static.lwn.net/images/ns/kernel/net-tx-mq.png)

Once again, the actual lists of outgoing packets normally exist in the form of special data structures in device-accessible memory. Once the device has these queues set up for it, the various policies associated with each class of service can be implemented. Each queue is managed independently, so more voice packets can be queued even if some other queue (background, say) is overflowing. 

David would appear to have worked hard to avoid creating trouble for network driver developers. Drivers for single-queue devices need not be changed at all, and the addition of multiqueue support is relatively straightforward. The first step is to replace the `alloc_etherdev()` call with a call to: 

```
struct net_device *alloc_etherdev_mq(int sizeof_priv, 
                                             unsigned int queue_count);
```

The new `queue_count` parameter describes the maximum number of transmit queues that the device might support. The actual number in use should be stored in the `real_num_tx_queues` field of the `net_device` structure. Note that this value can only be changed when the device is down. 

A multiqueue driver will get packets destined for any queue via the usual `hard_start_xmit()` function. To determine which queue to use, the driver should call: 

```
u16 skb_get_queue_mapping(struct sk_buff *skb);
```

The return value is an index into the array of transmit queues. One might well wonder how the networking core decides which queue to use in the first place. That is handled via a new `net_device` callback: 

```
u16	(*select_queue)(struct net_device *dev, struct sk_buff *skb);
```

The patch set includes an implementation of `select_queue()` which can be used with WME-capable devices. 

About the only other required change is for multiqueue drivers to inform the networking core about the status of specific queues. To that end, there is a new set of functions: 

```
struct netdev_queue *netdev_get_tx_queue(struct net_device *dev,
                                                 u16 index);
    
        void netif_tx_start_queue(struct netdev_queue *dev_queue);
        void netif_tx_wake_queue(struct netdev_queue *dev_queue);
        void netif_tx_stop_queue(struct netdev_queue *dev_queue);
```

A call to `netdev_get_tx_queue()` will turn a queue index into the `struct netdev_queue` pointer required by the other functions, which can be used to stop and start the queue in the usual manner. Should the driver need to operate on all of the queues at once, there is a set of helper functions: 

```
void netif_tx_start_all_queues(struct net_device *dev);
        void netif_tx_wake_all_queues(struct net_device *dev);
        void netif_tx_stop_all_queues(struct net_device *dev);
```

Naturally, there are a few other details to deal with, and the multiqueue interface is likely to evolve somewhat over time. At one point, David was hoping to have this feature ready for inclusion into 2.6.27, but that goal looks overly ambitious now. It does seem that [much of the ground work](<https://lwn.net/Articles/289154/>) will be merged in the next development cycle, though, meaning that full multiqueue support should be in good shape for merging in 2.6.28.
