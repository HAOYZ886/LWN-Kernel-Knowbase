---
title: "LPC: Making the net go faster"
url: https://lwn.net/Articles/458610/
date: "September 13, 2011"
category: "Networking-Congestion control"
author: "By Jonathan Corbet September 13, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 13, 2011

Almost every service offered by Google is delivered over the Internet, so it makes sense that the company would have an interest in improving how the net performs. The networking session at the 2011 Linux Plumbers Conference featured presentations from three Google developers, each of whom had a proposal for a significant implementation change. Between the three, it seems, there is still a lot of room for improvement in how we do networking. 

#### Proportional rate reduction

The "congestion window" is a TCP sender's idea of how much data it can have in flight to the other end before it starts to overload a link in the middle. Dropped packets are often a sign that the congestion window is too large, so TCP implementations normally reduce the window significantly when loss happens. Cutting the congestion window will reduce performance, though; if the packet loss was a one-time event, that slowdown will be entirely unnecessary. [RFC 3517](<https://tools.ietf.org/html/rfc3517>) describes an algorithm for bringing the connection up to speed quickly after a lost packet, but, Nandita Dukkipati says, we can do better. 

According to Nandita, a large portion of the network sessions involving Google's servers experience losses at some point; the ones that do can take 7-10 times longer to complete. RFC 3517 is part of the problem. This algorithm responds to a packet loss by immediately cutting the congestion window in half; that means that the sending system must, if the congestion window had been full at the time of the loss, wait for ACKs for half of the in-transit packets before transmitting again. That causes the sender to go silent for an extended period of time. It works well enough in simple cases (a single packet lost in a long-lasting flow), but it tends to clog up the works when dealing with short flows or extended packet losses. 

Linux does not use strict RFC 3517 now; it uses, instead, an enhancement called "rate halving." With this algorithm, the congestion window is not halved immediately. Once the connection goes into loss recovery, each incoming ACK (which will typically acknowledge the receipt of two packets at the other end) will cause the congestion window to be reduced by a single packet. Over the course of one full set of in-flight packets, the window will be cut in half, but the sending system will continue to transmit (at a lower rate) while that reduction is happening. The result is a smoother flow and reduced latency. 

But rate halving can be improved upon. The ACKs it depends on are themselves subject to loss; an extended loss can cause significant reduction of the congestion window and slow recovery. This algorithm also does not even begin the process of raising the congestion window back to the highest workable value until the recovery process is complete. So it can take quite a while to get back up to full speed. 

The proportional rate reduction algorithm takes a different approach. The first step is to calculate an estimate for the amount of data still in flight, followed by a calculation of what, according to the congestion control algorithm in use, the congestion window should now be. If the amount of data in the pipeline is less than the target congestion window, the system just goes directly into the TCP slow start algorithm to bring the congestion window back up. Thus, when the connection experiences a burst of losses, it will start trying to rebuild the congestion window right away instead of creeping along with a small window for an extended period. 

If, instead, the amount of data in flight is at least as large as the new congestion window, an algorithm similar to rate halving is used. The actual reduction is calculated relative to the new congestion window, though, rather than being a strict one-half cut. For both large and small losses, the emphasis on using estimates of the amount of in-flight data instead of counting ACKs is said to make recovery go more smoothly and to avoid needless reductions in the congestion window. 

How much more better is it? Nandita said that Google has been running experiments on some of its systems; the result has been a 3-10% reduction in average latency. Recovery timeouts have been reduced by 5%. This code is being deployed more widely on Google's servers; it also has been accepted for merging during the 3.2 development cycle. More information can be found in [this draft RFC](<https://tools.ietf.org/html/draft-mathis-tcpm-proportional-rate-reduction-01>). 

#### TCP fast open

Opening a TCP connection requires a three-packet handshake: a SYN packet sent by the client, a SYN-ACK response from the server, and a final ACK from the client. Until the handshake is complete, the link can carry no data, so the handshake imposes an unavoidable startup latency on every connection. But what would happen, asked Yuchung Cheng, if one were to send data with the handshake packets? For simple transactions - an HTTP GET request followed by the contents of a web page, for example - sending the relevant data with the handshake packets would eliminate that latency. The result of this thought is the "TCP fast open" proposal. 

[RFC 793](<https://www.ietf.org/rfc/rfc793.txt>) (describing TCP) does allow data to be passed with the handshake packets, with the proviso that the data not be passed to applications until the handshake completes. One can consider fudging that last requirement to speed the process of transmitting data through a TCP connection, but there are some hazards to be dealt with. An obvious problem is the amplification of SYN flood attacks, which are bad enough when they only involve the kernel; if each received SYN packet were to take up application resources as well, the denial of service possibilities would be significantly worse. 

Yuchung described an approach to fast open which is intended to get around most of the problems. The first step is the creation of a per-server secret which is hashed with information from each client to create a per-client cookie. That cookie is sent to the client as a special option on an ordinary SYN-ACK packet; the client can keep it and use it for fast opens in the future. The requirement to get a cookie first is a low bar for the prevention of SYN flood attacks, but it does make things a little harder. In addition, the server's secret is changed relatively often, and, if the server starts to see too many connections, fast open will simply be disabled until things calm down. 

One remaining problem is that about 5% of the systems on the net will drop SYN packets containing unknown options or data. There is little to be done in this situation; TCP fast open simply will not work. The client must thus remember cases where the fast-open SYN packet did not get through and just use ordinary opens in the future. 

Fast open will not happen by default; applications on both ends of the connection must specifically request it. On the client side, the `sendto()` system call is used to request a fast-open connection; with the new `MSG_FAST_OPEN` flag, it functions like the combination of `connect()` and `sendmsg()`. On the server side, a `setsockopt()` call with the `TCP_FAST_OPEN` option will enable fast opens. Either way, applications need not worry about dealing with the fast-open cookies and such. 

In Google's testing, TCP fast open has been seen to improve page load times by anything between 4% and 40%. This technique works best in situations where the round trip time is high, naturally; the bigger the latency, the more value there is in removing it. A patch implementing this feature will be submitted for inclusion sometime soon. 

#### Briefly: user-space network queues

While the previous two talks were concerned with improving the efficiency of data transfer over the net, Willem de Bruijn is concerned with network processing on the local host. In particular, he is working with high-end hardware: high-speed links, numerous processors, and, importantly, smart network adapters that can recognize specific flows and direct packets to connection-specific queues. By the time the kernel gets around to thinking about a given packet at all, it will already be sorted into the proper place, waiting for the application to ask for the data. 

Actual processing of the packets will happen in the context of the receiving process as needed. So it all happens in the right context and on the right CPU; intermediate processing at the software IRQ level will be avoided. Willem even described a new interface whereby the application would receive packets directly from the kernel via a shared memory segment. 

In other words, this talk described a variant of the network channels concept, where packet processing is pushed as close to the application as possible. There are numerous details to be dealt with, including the usual hangups for the channels idea: firewall processing and such. The proposed use of a file in sysfs to pass packets to user space also seems unlikely to pass review. But this work may eventually reach a point where it is generally useful; those who are interested can find the patches on [the unetq page](<https://code.google.com/p/kernel/wiki/unetq>).
