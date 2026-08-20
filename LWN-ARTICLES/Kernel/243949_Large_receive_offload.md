---
title: Large receive offload
url: https://lwn.net/Articles/243949/
date: "August 1, 2007"
category: "Device drivers-Network drivers; Networking"
author: "By Jonathan Corbet August 1, 2007"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 1, 2007

High-performance networking is continually faced with a challenge: local networking technologies are getting faster more quickly than processor and memory speeds. So every time that the venerable Ethernet technology provides another speed increment, networking developers must find ways to enable the rest of the system to keep up - even on fast contemporary hardware. 

One recurring idea is to push more of the work into the networking hardware itself. TCP offload engines have been around since the days when systems were having trouble keeping up with 10Mb Ethernet, but that technology has always been limited in its acceptance; see [this 2005 LWN article](<http://lwn.net/Articles/148697/>) for some discussion of why. But some more restrained hardware assist techniques have been more successful; for example, TCP segmentation offload (TSO), where network adapters turn a stream of data into packets for transmission, is well supported under Linux. 

Use of TSO can boost networking performance considerably. When one is dealing with thousands of packets every second, even a slight per-packet assist will add up. TSO reduces the amount of work needed to build headers and checksum the data, and it cuts down on the number of times that the driver must program operations into the network adapter. There is, however, no analogous assistance for incoming data. So, if you have two identical Linux servers with one sending a high-bandwidth stream to the other, the receiving side may be barely keeping up with the load while the transmitting side barely breaks a sweat. 

Proposals for assistance for packet reception often go under the name "large receive offload" (LRO); the idea was first proposed for Linux in [this OLS 2005 talk [PDF]](<http://www.linuxinsight.com/files/ols2005/grossman-reprint.pdf>). The initial LRO implementation used hardware features found in Neterion adapters; it never made it into the mainline and little has been heard from that direction since. The LRO idea has recently returned, though, in the form of [this patch by Jan-Bernd Themann](<http://lwn.net/Articles/243950/>). Interestingly, the new LRO code does not require any hardware assistance at all. 

With Jan-Bernd's patch, a driver must, to support LRO, fill in an LRO manager structure which looks like this: 

```
#include <linux/inet_lro.h>
    
        struct net_lro_mgr {
    	struct net_device *dev;
    	struct net_lro_stats stats;
    
    	unsigned long features;
    
    	u32 ip_summed; /* Options to be set in generated SKB in page mode */
    	int max_desc; /* Max number of LRO descriptors  */
    	int max_aggr; /* Max number of LRO packets to be aggregated */
    
    	struct net_lro_desc *lro_arr; /* Array of LRO descriptors */
    	/*
    	 * Optimized driver functions
    	 *
    	 * get_skb_header: returns tcp and ip header for packet in SKB
    	 */
    	int (*get_skb_header)(struct sk_buff *skb, void **ip_hdr,
    			      void **tcpudp_hdr, u64 *hdr_flags, void *priv);
    
    	/*
    	 * get_frag_header: returns mac, tcp and ip header for packet in SKB
    	 *
    	 * @hdr_flags: Indicate what kind of LRO has to be done
    	 *             (IPv4/IPv6/TCP/UDP)
    	 */
    	int (*get_frag_header)(struct skb_frag_struct *frag, void **mac_hdr,
    			       void **ip_hdr, void **tcpudp_hdr, u64 *hdr_flags,
    			       void *priv);
        };
```

In this structure, `dev` is the network interface for which LRO is to be implemented; `stats` contains some statistics which can be queried to see how well LRO is working. The `features` field controls how the LRO code should feed packets into the networking stack; it has two flags defined currently: 

  * `LRO_F_NAPI` says that the driver is [NAPI](<http://lwn.net/Articles/30107/>) compliant, and that, in particular, packets should be passed upward with `netif_receive_skb()`. 

  * `LRO_F_EXTRACT_VLAN_ID` is for drivers with VLAN support. This article won't go further into VLAN support for the simple reason that your editor does not understand it. 

Checksum information for the final packets should go into `ip_summed`. The maximum number of "LRO descriptors" should be stored in `max_desc`. Each descriptor describes one TCP stream, so the maximum limits the number of streams for which LRO can be done simultaneously. Increasing the maximum requires more memory and will slow things a bit, since packets are matched to streams by way of a linear search. `max_aggr` is the maximum number of incoming packets which will be aggregated into a single, larger packet. The `lro_arr` array contains the descriptors for tracking streams; the driver should provide it with enough memory for at least `max_desc` structures or very unpleasant things are likely to happen. 

Finally, there are the `get_skb_header()` and `get_frag_header()` methods. Their job is to locate the IP and TCP headers in a packet as quickly as possible. Typically a driver will only provide one of the two functions, depending on how it feeds packets into the LRO aggregation code. 

A driver which receives packets in fully-completed `sk_buff` structures would normally pass them up directly to the network stack with `netif_rx()` or `netif_receive_skb()`. If LRO is being done, instead, the packets should be handed to: 

```
void lro_receive_skb(struct net_lro_mgr *lro_mgr,
    		     	 struct sk_buff *skb,
    		     	 void *priv);
```

This function will attempt to identify an LRO descriptor for the given packet, creating one if need be. Then it will try to join that packet with any others in the stream, making one large, fragmented packet. In the process, it will call the driver's `get_skb_header()` method, passing through the pointer given as `priv`. If the packet cannot be aggregated with others (it may not be a TCP packet, for example, or it could have TCP options which require it to be processed separately) it will be passed directly to the network stack. Either way, the driver can consider it delivered and move on to its next task. 

Some drivers receive packets directly into memory represented by `page` structures, constructing the full `sk_buff` structure after reception. For such drivers, the interface is: 

```
void lro_receive_frags(struct net_lro_mgr *lro_mgr,
    	 	       	   struct skb_frag_struct *frags,
    			   int len, int true_size, 
    			   void *priv, __wsum sum);
```

The LRO code will build the necessary `sk_buff` structure, perhaps aggregating fragments from several packets, and (sooner or later) feed the results to the network stack. It will call the driver's `get_frag_header()` method to locate the headers in the first element of the `frags` array; that method should also ensure that the packet is an IPv4 TCP packet and set `LRO_IPV4` and `LRO_TCP` in the `flags` argument if so. 

Combined packets will be pushed up into the network stack whenever `max_aggr` individual packets have been merged into them. Delaying data for too long while waiting for additional packets is not a good idea, though; occasionally packets should be sent on even if they are not as large as they could be. The function for this job is: 

```
void lro_flush_all(struct net_lro_mgr *lro_mgr);
```

It will cause all packets to sent on. A logical place for such a call might be at the end of a NAPI driver's `poll()` method. An individual stream can be flushed with: 

```
void lro_flush_pkt(struct net_lro_mgr *lro_mgr,
    		       struct iphdr *iph, 
    		       struct tcphdr *tcph);
```

This call will locate the stream associated with the given IP and TCP headers and send its accumulated data onward. It will _not_ add any data associated with the given headers; the packet associated with those headers should have already been added with one of the receive functions if need be. 

That is, for all practical purposes, the entire interface. One might well wonder how this code can improve performance, given that it is just aggregating packets which have already been received in the usual way by the driver. The answer is that it is reducing the number of packets that the network stack has to work with, cutting the per-packet overhead at higher levels in the stack. A clever driver can, using the `struct page` approach, also reduce the number of memory allocations required for each packet, which can be a big win. So LRO appears to be worth having, and current plans call for it to be merged in 2.6.24.
