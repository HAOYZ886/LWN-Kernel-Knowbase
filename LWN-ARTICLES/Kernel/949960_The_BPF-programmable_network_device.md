---
title: "The BPF-programmable network device"
url: https://lwn.net/Articles/949960/
date: "November 6, 2023"
category: "BPF-Device drivers; Networking-Performance"
author: "By Jonathan Corbet November 6, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 6, 2023

Containers and virtual machines on Linux communicate with the world via virtual network devices. This arrangement makes the full power of the Linux networking stack available, but it imposes the full overhead of that stack as well. Often, the routing of this networking traffic can be handled with relatively simple logic; the BPF-programmable network device, which was merged for the 6.7 kernel release, makes it possible to avoid expensive network processing, in at least some cases. 

When a guest (either a container or a virtual machine) sends data over the network in current systems, that data first enters the network stack within that guest, where it is formed into packets and sent out through the virtual interface. On the host side, that packet is received and handled, once again within the network stack. If the packet is destined for a peer outside of the host, the packet will be routed to a (real) network interface for retransmission. The guest's data has made it into the world, but only after having passed through two network stacks. 

The new device, named "netkit", aims to short out some of that overhead. It is, in some sense, a typical virtual device in that a packet transmitted at one end will only pass through the host system's memory before being received at the other. The difference is in how transmission works. Every network-interface driver provides a [`net_device_ops` structure](<https://elixir.bootlin.com/linux/v6.6/source/include/linux/netdevice.h#L1057>) containing a large number of function pointers — as many as 90 in the 6.6 kernel. One of those is `ndo_start_xmit()`: 

```
netdev_tx_t	(*ndo_start_xmit)(struct sk_buff *skb, struct net_device *dev);
```

This function's job is to initiate the transmission of the packet found in `skb` by way of the indicated device `dev`. In a typical virtual device, this function will immediately "receive" the packet into the network stack on the peer side with a call to a function like [`netif_rx()`](<https://elixir.bootlin.com/linux/v6.6/source/net/core/dev.c#L5108>). The netkit device, though, behaves a bit differently. 

When this virtual interface is set up, it is possible to load one or more BPF programs into each side of the interface. Since netkit BPF programs can affect traffic routing on the host side, only the host is allowed to load these programs for either the host or the guest. The `ndo_start_xmit()` callback provided by netkit will, rather than just passing the packet back into the network stack, invoke each of the attached programs in sequence, passing the packet to each. The BPF programs are able to modify the packet (to change the destination device, for example), and are expected to return a value saying what should be done next: 

  * `NETKIT_NEXT`: continue processing with the next BPF program in the series (if any). If there are no more programs to invoke, this return is treated like `NETKIT_PASS`. 
  * `NETKIT_PASS`: immediately pass the packet into the receiving side's network stack without calling any other BPF programs. 
  * `NETKIT_DROP`: immediately drop the packet. 
  * `NETKIT_REDIRECT`: immediately redirect the packet to a new network device, queuing it for transmission without the need to pass through the host's network stack. 

Each interface can be configured with a default policy (either `NETKIT_PASS` or `NETKIT_DROP`) that applies if there is no BPF program loaded to make the decision. Most of the time, the right policy is probably to drop the packet, ensuring that no traffic leaks out of the guest until the interface is fully configured to handle it. 

There are performance gains to be had if the decision to drop a packet can be made as soon as possible. Unwanted network traffic can often come in great quantities, so the less time spent on it, the better. But, as [the changelog](<https://git.kernel.org/linus/35dfaad7188c>) states, the best performance gains may come from the ability to redirect packets without re-entering the network stack: 

> For example, if the BPF program determines that the skb must be sent out of the node, then a redirect to the physical device can take place directly without going through per-CPU backlog queue. This helps to shift processing for such traffic from softirq to process context, leading to better scheduling decisions/performance. 

According to the [slides](<http://vger.kernel.org/bpfconf2023_material/tcx_meta_netdev_borkmann.pdf>) from a 2023 Linux Storage, Filesystem, Memory-Management and BPF Summit talk, guests operating through the netkit device (which was called "meta" at that time) are able to attain TCP data-transmission rates that are just as high as can be had by running directly on the host. The performance penalty for running within a guest has, in other words, been entirely removed. 

Given the potential performance gains for some users, it's not surprising that this patch series, posted by Daniel Borkmann but also containing work by Nikolay Aleksandrov, was merged quickly. It was first [posted](<https://lwn.net/ml/bpf/20230926055913.9859-1-daniel@iogearbox.net/>) to the BPF mailing list on September 26, went through four revisions there, then applied for the 6.7 merge window one month later. This feature will not be for all users but, for those who are deploying network-intensive applications within containers or virtual machines, it could be appealing indeed.
