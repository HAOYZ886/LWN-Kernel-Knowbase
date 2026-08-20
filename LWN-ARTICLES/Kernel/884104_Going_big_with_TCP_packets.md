---
title: Going big with TCP packets
url: https://lwn.net/Articles/884104/
date: "February 14, 2022"
category: "Networking-IPv6"
author: "By Jonathan Corbet February 14, 2022"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 14, 2022

Like most components in the computing landscape, networking hardware has grown steadily faster over time. Indeed, today's high-end network interfaces can often move data more quickly than the systems they are attached to can handle. The networking developers have been working for years to increase the scalability of their subsystem; one of the current projects is [the BIG TCP patch set](<https://lwn.net/ml/netdev/20220203015140.3022854-1-eric.dumazet@gmail.com/>) from Eric Dumazet and Coco Li. BIG TCP isn't for everybody, but it has the potential to significantly improve networking performance in some settings. 

Imagine, for a second, that you are trying to keep up with a 100Gb/s network adapter. As networking developer Jesper Brouer [described](<https://lwn.net/Articles/629155/>) back in 2015, if one is using the longstanding maximum packet size of 1,538 bytes, running the interface at full speed means coping with over eight-million packets per second. At that rate, CPU has all of about 120ns to do whatever is required to handle each packet, which is not a lot of time; a single cache miss can ruin the entire processing-time budget. 

The situation gets better, though, if the number of packets is reduced, and that can be achieved by making packets larger. So it is unsurprising that high-performance networking installations, especially local-area networks where everything is managed as a single unit, use larger packet sizes. With proper configuration, packet sizes up to 64KB can be used, improving the situation considerably. But, in settings where data is being moved in units of megabytes or gigabytes (or more — cat videos are getting larger all the time), that still leaves the system with a lot of packets to handle. 

Packet counts hurt in a number of ways. There is a significant fixed overhead associated with every packet transiting a system. Each packet must find its way through the network stack, from the upper protocol layers down to the device driver for the interface (or back). More packets means more interrupts from the network adapter. The `sk_buff` structure ("SKB") used to represent packets within the kernel is a large beast, since it must be able to support just about any networking feature that may be in use; that leads to significant per-packet memory use and memory-management costs. So there are good reasons to wish for the ability to move data in fewer, larger packets, at least for some types of applications. 

The length of an IP packet is stored in the IP header; for both IPv4 and IPv6, that length lives in a 16-bit field, limiting the maximum packet size to 64KB. At the time these protocols were designed, a 64KB packet could take multiple seconds to transmit on the backbone Internet links that were available, so it must have seemed like a wildly large number; surely 64KB would be more than anybody would ever rationally want to put into a single packet. But times change, and 64KB can now seem like a cripplingly low limit. 

Awareness of this problem is not especially recent: there is a solution (for IPv6, at least) to be found in [RFC 2675](<https://datatracker.ietf.org/doc/html/rfc2675>), which was adopted in 1999. The IPv6 specification allows the placement of ["hop-by-hop" headers](<https://en.wikipedia.org/wiki/IPv6_packet#Hop-by-hop_options_and_destination_options>) with additional information; as the name suggests, a hop-by-hop header is used to communicate options between two directly connected systems. RFC 2675 enables larger packets with a couple of tweaks to the protocol. To send a "jumbo" packet, a system must set the (16-bit) IP payload length field to zero and add a hop-by-hop header containing the real payload length. The length field in that header is 32 bits, meaning that jumbo packets can contain up to 4GB of data; that should surely be enough for everybody. 

The BIG TCP patch set adds the logic necessary to generate and accept jumbo packets when the maximum transmission unit (MTU) of a connection is set sufficiently high. Unsurprisingly, there were a number of details to manage to make this actually work. One of the more significant issues is that packets of any size are rarely stored in physically contiguous memory, which tends to be hard to come by in general. For zero-copy operations, where the buffers live in user space, packets are guaranteed to be scattered through physical memory. So packets are represented as a set of "fragments", which can be as short as one (4KB) page each; network interfaces handle the task of assembling packets from fragments on transmission (or fragmenting them on receipt). 

Current kernels limit the number of fragments stored in an SKB to 17, which is sufficient to store a 64KB packet in single-page chunks. That limit will clearly interfere with the creation of larger packets, so the patch set raises the maximum number of fragments (to 45). But, as Alexander Duyck [pointed out](<https://lwn.net/ml/netdev/ee1fedeb33cd989379b72faac0fd6a366966f032.camel@gmail.com/>), many interface drivers encode assumptions about the maximum number of fragments that a packet may be split into. Increasing that limit without fixing the drivers could lead to performance regressions or even locked-up hardware, he said. 

After some discussion, Dumazet [proposed](<https://lwn.net/ml/netdev/CANn89iKMa5fT3HhKdO2K=WFxwBsRBr_HN=sxPDqX-MJvWyoz5Q@mail.gmail.com/>) working around the problem by adding a configuration option controlling the maximum number of allowed fragments for any given packet. That is fine for sites that build their own kernels, which prospective users of this feature are relatively likely to do. It offers little help for distributors, though, who must pick a value for this option for all of their users. 

In any case, many drivers will need to be updated to handle jumbo packets. Modern network interfaces perform segmentation offloading, meaning that much of the work of creating individual packets is done within the interface itself. Making segmentation offloading work with jumbo packets tends to involve a small number of tweaks; a few drivers are updated in the patch set. 

One other minor problem has to do with the placement of the RFC 2675 hop-by-hop header. These headers, per the IPv6 standard, are placed immediately after the IP header; that can confuse software that "knows" that the TCP header can be found immediately after the IP header in a packet. The `tcpdump` utility has some problems in this regard; it also seems that there are a fair number of BPF programs in the wild that contain this assumption. For this reason, jumbo-packet handling is disabled by default, even if the underlying hardware and link could handle those packets. 

Dumazet included some brief benchmark results with the patch posting. Enabling a packet size of 185,000 bytes increased network throughput by nearly 50% while also reducing round-trip latency significantly. So BIG TCP seems like an option worth having, at least in the sort of environments (data centers, for example) that use high-speed links and can reliably deliver large packets. If tomorrow's cat videos arrive a little more quickly, BIG TCP may be part of the reason. 

See [Dumazet's 2021 Netdev talk](<https://netdevconf.info/0x15/session.html?BIG-TCP>) on this topic for more details.
