---
title: "Transport-level protocols in user space"
url: https://lwn.net/Articles/691887/
date: "June 20, 2016"
category: Networking
author: "By Jonathan Corbet June 20, 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 20, 2016

The Linux networking developers have long held a strong opinion about user-space protocol implementations: they should be avoided in favor of making the in-kernel implementation better. So it might be surprising to see a veteran networking developer post a patch set aimed at making user-space implementations easier. A look at this patch and its motivations shines an interesting light on changes that are taking place in the networking world. 

The developer in question is Tom Herbert; the patch set is titled "[transports over UDP](<https://lwn.net/Articles/691873/>)", or "TOU". This patch set allows any transport-layer protocol (such as TCP or [SCTP](<https://en.wikipedia.org/wiki/Stream_Control_Transmission_Protocol>)) to be packaged up and carried within a stream of UDP packets. The mechanism by which this is done is described in [this draft RFC](<https://tools.ietf.org/html/draft-herbert-transports-over-udp-00>) (also written by Tom); it uses the draft [generic UDP encapsulation (GUE)](<https://tools.ietf.org/html/draft-ietf-nvo3-gue-01>) specification for the actual encapsulation. The patch set adds basic (still incomplete) TOU support to the kernel, with the ability to encapsulate TCP connections in particular. 

Any destination that is reachable via UDP should also, in principle, be reachable directly using a transport-layer protocol, so one might well wonder why it is worth the trouble to hide that protocol inside UDP. The fact that this encapsulation comes with (according to Tom's benchmark numbers) a 10-15% performance loss also serves to increase eyebrow elevation. It turns out, though, that there are a few advantages to encapsulating a protocol like TCP in this way. 

The first of those is that, as mentioned above, it facilitates running the transport-layer protocol stack in user space. The usual reason for bypassing the kernel in favor of user-space networking is to maximize performance for specific types of communications, but that is not the intent here. Tom (and his employer Facebook) envision putting the transport-layer protocol code into desktop and mobile client applications instead, and the main reason to do that is to speed progress on the development and adoption of newer protocol versions. Tom [described the purpose](<https://lwn.net/Articles/691888/>) this way: 

The major problem we have is that Android and to some extent iOS & Windows take a long time to update the kernel, and it can take an _extremely_ long time if we need them to actively enable features that are needed by applications. For instance, TFO [[TCP fast open](<https://lwn.net/Articles/508865/>)] was put in the Linux [kernel] several years ago, but it still hasn't been enabled in Android and only fairly recently enabled in iOS. If we (e.g. Facebook) implement a userspace stack in clients and control the stack in our servers we can roll out a feature like that in a couple of months. 

This is, in other words, an attempt to address the [protocol ossification problem](<https://lwn.net/Articles/667059/>) by taking more direct control of the protocol code on both sides of the connection. That will, it is thought, allow network protocols to start evolving at a reasonable pace again — when users are running the right user-space implementation, at least. 

The other benefit to TOU is related to the protocol ossification problem, but also to the privacy and security concerns that are prevalent on the net. Once the transport-layer headers (and payload) are encapsulated within a UDP packet, they no longer need to be sent in the clear. The TOU draft specifies that DTLS ([datagram transport layer security](<https://tools.ietf.org/html/rfc4347>)) can be used to encrypt the payload of the UDP packets, hiding the transport-layer information entirely from any system other than the endpoints. It thus brings back the idea of end-to-end networking, where the endpoints know about a specific communication and all other machines in the middle exist only to serve as dumb transport. Eavesdropping on communications — or even collecting metadata — would be much harder in a world where TOU was widely deployed and used. 

TOU can thus provide privacy, but there is another advantage: routers along the path ("middleboxes") will be unable to inspect packets and filter out specific protocols. This filtering makes the addition of any new protocol difficult at best. Adding a simple flag for [explicit congestion notification (ECN)](<https://en.wikipedia.org/wiki/Explicit_Congestion_Notification>) made much of the net unreachable; the developers of [multipath TCP](<https://lwn.net/Articles/544399/>) have had to go to considerable lengths to disguise the new protocol as boring vanilla TCP to get it past middleboxes. But once a middlebox loses visibility into the protocol that is actually being used, it also loses the ability to filter it. That too, Tom hopes, will fight ossification and speed the adoption of new protocols. 

It is also worth noting that encapsulating a TCP connection in this way separates it from the underlying network, making the endpoints of TCP connections mobile. That, too, could be useful in a world where computers move rapidly from one network to the next. 

Before all this can happen, though, the TOU patches need to find their way into the kernel. Networking maintainer David Miller [did not reject them](<https://lwn.net/Articles/691889/>), but he did express some strong misgivings about the whole thing, starting with the additional maintenance burden that it implies: 

We also now have to debug against every single userland TCP implementation someone can come up with, the barrier to entry is insanely low with TOU. Maybe this sounds great to you, but to me it is quite terrifying This has a monumental impact upon maintenance of our TCP stack. 

He suggested that, in the future, developers would treat TOU-related bug reports the way reports involving proprietary modules are treated now — by ignoring them. He also worried that the middlebox problem might not be quite so easily solved: 

This sounds really great on paper, however I find it hard to believe that makers of middleware boxes are just going to throw their hands in the air and say "oh well, we lose." Rather, I think people are going to start adding rules to block TOU tunnels entirely because they cannot inspect nor conditionally filter/rewrite the contents. 

Tom responded that the bulk of the net allows UDP traffic now, and that a number of higher-level protocols, including [QUIC](<https://lwn.net/Articles/558826/>), have been successfully deployed. The nascent [path layer UDP substrate](<https://www.ietf.org/mail-archive/web/spud/current/msg00501.html>) effort is working to improve flow control for UDP-based transport protocols. Overall, that seems to be the direction the net is taking, and TOU should fit into it well, so it may not be unreasonable to believe that it will get through middleboxes in the future. 

It would seem that the way to save the Internet is to take the higher-level protocols out of the kernel's hands and bury them, instead, in the applications. There is one potential problem here, though, that has not yet really been discussed on the list. TOU _could_ be used to achieve rapid deployment of new Internet standards. Or it could be used to deploy a whole new set of proprietary transport-level protocols that are not easily accessible outside of closed-source applications. It could be that TOU-like technologies (including the already-deployed QUIC) will, instead, retard the adoption of open Internet standards by rendering them irrelevant. Tom [acknowledged](<https://lwn.net/Articles/692062/>) that the whole idea ""conflicts with our idea of what the architecture of the Internet should be"", but also asserted that the development community cannot ignore it. 

In summary, enabling this mode might lead to a world where the network evolves more quickly and is more resistant to surveillance and eavesdropping. On the other hand, it could lead to a fragmented net where each large company has its own transport-layer protocols baked into its own proprietary applications. Guarantees that things would go the right way would appear to be in short supply; this area is going to be interesting to watch.
