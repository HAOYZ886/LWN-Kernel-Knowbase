---
title: QUIC as a solution to protocol ossification
url: https://lwn.net/Articles/745590/
date: "January 29, 2018"
category: "Networking-Protocols"
author: "By Jonathan Corbet January 29, 2018 linux.conf.au"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 29, 2018

* * *

[linux.conf.au](<https://lwn.net/Archives/ConferenceByYear/#2018-linux.conf.au>)

The TCP protocol has become so ubiquitous that, to many people, the terms "TCP/IP" and "networking" are nearly synonymous. The fact that introducing new protocols (or even modifying existing protocols) [has become nearly impossible](<https://lwn.net/Articles/667059/>) tends to reinforce that situation. That is not stopping people from trying, though. At linux.conf.au 2018, Jana Iyengar, a developer at Google, discussed the current state of the QUIC protocol which, he said, is now used for about 7% of the traffic on the Internet as a whole. 

QUIC ("quick UDP Internet connection") is, for now, intended for situations where the HTTP transport protocol is used over TCP. It has been under development for several years (LWN first [looked at it](<https://lwn.net/Articles/558826/>) in 2013), and was first deployed at Google in 2014. The main use for QUIC now is to move data between Google services and either the Chrome browser or various mobile apps. Using QUIC causes a 15-18% drop in rebuffering in YouTube and a 3.6-8% drop in Google search latency, Iyengar said. Getting that kind of improvement out of applications that have already been aggressively optimized is "somewhat absurd". 

Use of QUIC increased slowly during 2015 before suddenly dropping to zero in December. It seems that somebody found a bug that could result in some requests being transmitted unencrypted, so QUIC was shut down until the issue could be fixed. In August 2016, usage abruptly doubled when QUIC was enabled in the YouTube app on phones. If anybody ever doubted that mobile is the future of computing, he said, this should convince them otherwise. Summed up, 35% of Google's outbound traffic is carried over QUIC now. 

The standard network stack, as used for the world-wide web, employs HTTP on top of the TLS cryptographic layer which, in turn, sits on top of TCP. QUIC replaces those components with a new protocol based on the UDP datagram protocol. From that base, QUIC builds a reliable connection-oriented protocol, complete with TCP-like congestion-control features. There is support for both encryption and HTTP within QUIC; it can combine the cryptographic and HTTP handshakes into a single packet. 

Thus far, both development and deployment of QUIC have been done primarily by Google. An IETF working group was formed to standardize the protocol in 2016, though. Among other things, standardization will replace the current QUIC cryptographic layer with one based on TLS 1.3 which, Iyengar said, took a number of its ideas from the current QUIC implementation. 

#### Accelerating HTTP

A typical web page has a long list of objects (HTML, CSS, images, etc.) that must be loaded from the server. The HTTP/1.x protocol only allows for a single object to be transmitted at a time; that can be a problem when a large object, which takes a long time to transmit, blocks the transmission of many other objects. This problem, referred to as [![\[Jana Iyengar\]](https://static.lwn.net/images/conf/2018/lca/JanaIyengar-sm.jpg)](<https://lwn.net/Articles/745595/>) "head-of-line blocking", increases the time it takes to present a usable web page to the reader. Implementations using HTTP/1.x tend to work around head-of-line blocking by establishing multiple connections in parallel, which has its own problems. Those connections are relatively expensive, compete with each other, and cannot be managed together by congestion-control algorithms and the like. 

HTTP/2 was designed to address this problem using multiple "streams" built into a single connection. Multiple objects can be sent over a stream in parallel by multiplexing them into the connection. That helps, but it creates a new problem: the loss of a single packet will stall transmission of all of the streams at once, creating new latency issues. This variant on the head-of-line-blocking problem is built into TCP itself and cannot be fixed with more tweaks at the HTTP level. 

TCP suffers other problems as well. Its connection setup latency, involving a three-way handshake, is relatively high. Latency is a critical part of a user's experience with a web service, and the setup latency in TCP can be a significant part of that latency. Middleboxes (routers between the endpoints of a connection) interfere with traffic and make it difficult to improve the protocol. They aren't supposed to be looking at TCP headers, but they do so anyway and make decisions based on what they see, often blocking traffic that looks in any way out of the norm. This "ossification" of the protocol makes it nearly impossible to make changes to TCP itself. For example, [TCP fast open](<https://lwn.net/Articles/508865/>) has been available in the Linux kernel (and others) for years, but still is not really deployed because middleboxes will not allow it. 

QUIC tries to resolve a number of these issues. The first time two machines talk over QUIC, a single round-trip is enough to establish the connection. For subsequent connections, cached information can be used to reduce that number to zero; the connection packet can be followed immediately by the request itself. HTTP streams map directly onto streams implemented in QUIC; a packet loss in one stream will not impact the others. The end result is the elimination of many sources of latency in typical interactions over the net. 

#### Requirements, metrics, and implementations

The QUIC developers set out to create a protocol that was both deployable and evolvable. That dictated the use of UDP, which is able to get through middleboxes with a minimum of interference. UDP also facilitates the creation of a user-space implementation, which was also desired. (Iyengar didn't say this, but one reason to want such an implementation is to get the protocol deployed and updated quickly; many systems out there rarely receive kernel updates.) Low-latency connection establishment was a requirement, as was stream multiplexing. Beyond that, there was a desire for more flexible congestion control. This sort of work can (and has been) done in the Linux kernel, but the bar for inclusion there is high. The QUIC developers wanted to be able to experiment with various algorithms and see how they worked. 

One other important requirement was resilience to "NAT rebinding". Most connections onto the Internet go through a network-address translation (NAT) box that hides the original request and port information. For TCP connections, the NAT box can see the SYN and FIN packets and know when a particular binding can be taken down. UDP itself has no "connection" concept, so NAT boxes carrying UDP traffic cannot associate it with a connection created by a higher-level protocol like QUIC. They thus have no indication of when a connection is no longer in use and instead have to rely on timers to decide when to tear down a specific port binding. As a result, UDP port bindings can be taken down while the QUIC connection using them is still active. The next UDP packet associated with that connection will cause a new binding to be established; that will cause the traffic to suddenly appear to be coming from a different port. QUIC packets must thus include the information needed to detect and handle such rebindings. 

A member of the audience asked why QUIC was implemented over UDP rather than directly on top of IP. Iyengar pointed to the [SCTP](<https://en.wikipedia.org/wiki/Stream_Control_Transmission_Protocol>) protocol as an example of the problem with new IP-based protocols: it comes down to the middleboxes again. SCTP has been around for years, but middleboxes still do not recognize it and tend to block it. As a result, SCTP cannot be reliably used on the net. Actually deploying a new IP-based protocol, he said, is simply impossible on today's Internet. Additionally, working on top of UDP makes a user-space implementation easier. 

As noted above, deployment of QUIC has led to significant improvements in performance for Google services. The significant drop in search latency is mainly a result of eliminating round trips during connection setup. As a result, it tends to show the biggest improvement for users on slow networks. A search done in South Korea is likely to show a 1.3% improvement in latency, but in India that improvement is over 13%. Iyengar said that people measurably spent more time watching more videos when they are doing so over QUIC; that was presented as a good thing. 

One key feature of QUIC is that the transport headers — buried inside the UDP packets — are encrypted. Beyond the obvious privacy benefits, encryption prevents ossification of the protocol by middleboxes, which can't make routing decisions based on information they can't understand. A few things have to be transmitted in clear text, though; a connection ID is required, for example, to find the key needed to decrypt the rest. The first byte of the clear data was a flags field which, he said, was promptly ossified by a middlebox vendor, leading to packets being dropped when a new flag was set. 

That was a classic example of why changing network protocols is hard and what needs to be done to improve the situation. Middleboxes are the control points for the Internet as we know it now. The only defense against ossification of network protocols by middleboxes, he said at the conclusion of the talk, is encryption. 

There were some questions from the audience regarding implementations. Most of them are still a work in progress, he said. The [quic-go](<https://github.com/lucas-clemente/quic-go>) implementation is coming along. There are implementations being done by Apple and Microsoft, and a certain amount of interoperability testing has been done on those. When asked about an open-source reference implementation in particular, Iyengar pointed to the chromium browser, which is open-source. Other implementations exist, but everybody is waiting for the IETF process to finish. 

[The video of this talk](<https://www.youtube.com/watch?v=BazWPeUGS8M>) is available. 

[Your editor would like to thank the Linux Foundation and linux.conf.au for assisting with his travel to this event.]
