---
title: "Multipath TCP: an overview"
url: https://lwn.net/Articles/544399/
date: "March 26, 2013"
category: "Networking-Protocols"
author: "By Jonathan Corbet March 26, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 26, 2013

The world was a simpler place when the TCP/IP network protocol suite was first designed. The net was slow and primitive and it was often a triumph to get a connection to a far-away host at all. The machines at either end of a TCP session normally did not have to concern themselves with how that connection was made; such details were left to routers. As a result, TCP is built around the notion of a (single) connection between two hosts. The [Multipath TCP (MPTCP) project](<http://multipath-tcp.org>) looks to change that view of networking by adding support for multiple transport paths to the endpoints; it offers a lot of benefits, but designing a deployable protocol for today's Internet is surprisingly hard. 

Things have gotten rather more complicated in the years since TCP was first deployed. Connections to multiple networks, once the province of large server systems, are now ubiquitous; a smartphone, for example, can have separate, simultaneous interfaces to a cellular network, a WiFi network, and, possibly, other networks via Bluetooth or USB ports. Each of those networks provides a possible way to reach a remote host, but any given TCP session will use only one of them. That leads to obvious policy considerations (which interface should be used when) and operational difficulties: most handset users are familiar with how a WiFi-based TCP session will be broken if the device moves out of range of the access point, for example. 

What if a TCP session could make use of all of the available paths between the two endpoints at any given time? There would be performance improvements, since each of the paths could carry data in parallel, and congested paths could be avoided in favor of faster paths at any given time. Sessions could also be more robust. Imagine a video stream that is established over both WiFi and cellular networks; if the watcher leaves the house (one hopes somebody else is driving), the stream would shift transparently to the cellular connection without interruption. Data centers, where multiple paths between systems and variable congestion are both common, could also make use of a multipath-capable transport protocol. 

The problem is that TCP does not work that way. Enter MPTCP, which _is_ designed to work that way. 

#### How it works

A TCP session is normally set up by way of a three-way handshake. The initiating host sends a packet with the SYN flag set, the receiving host, if it is amenable to the connection, responds with a packet containing both the SYN and ACK flags. The final ACK packet sent by the initiator puts the connection into the "established" state; after that, data can be transferred in either direction. 

An MPTCP session starts in the same way, with one change: the initiator adds the new MP_CAPABLE option to the SYN packet. If the receiving host supports MPTCP, it will add that option to its SYN-ACK reply; the two hosts will also include cryptographic keys in these packets for later use. The final ACK (which must also carry the MP_CAPABLE option) establishes a multipath session, albeit a session using a single path just like traditional TCP. 

When MPTCP is in use, both sides recognize a distinction between the session itself and any specific "subflow" used by that session. So, at any point, either party to the session can initiate another TCP connection to the other side, with the proviso that the address and/or port at one end or the other of the connection must differ. So, if a smartphone has initiated an MPTCP connection to a server using its WiFi interface: 

> ![\[Cheesy diagram\]](https://static.lwn.net/images/2013/mptcp1.png)

It can add another subflow at any time by connecting to the same server by way of its cellular interface: 

> ![\[Cheesy diagram\]](https://static.lwn.net/images/2013/mptcp2.png)

That subflow is added by sending a SYN packet with the MP_JOIN option; it also includes information on which MPTCP session is to be joined. Needless to say, the protocol designers are concerned that a hostile party might try to join somebody else's session; the previously-exchanged cryptographic keys are used to prevent such attacks from succeeding. If the receiving server is amenable to adding the subflow, it will allow the establishment of the new TCP connection and add it to the MPTCP session. 

Once a session has more than one subflow, it is up to the systems on each end to decide how to split traffic between them (though it is possible to mark a specific subflow for use only when any others no longer work). A single receive window applies to the session as a whole. Each subflow looks like a normal TCP connection, with its own sequence numbers, but the session as a whole has a separate sequence number; there is another TCP option (DSS, or "Data Sequence Signal") which is used to inform the other end how data on each subflow fits into the overall stream. 

Subflows can come and go over the life of an MPTCP connection. They can be explicitly closed by either end, or they can simply vanish if one of the paths becomes unavailable. If the underlying machinery is working well, applications should not even notice these changes. Just like IP can hide routing changes, MPTCP can hide the details of which paths it is using at any given time. It should, from an application's point of view, just work. 

Needless to say, there are vast numbers of details that have been glossed over here. Making a protocol extension like this work requires thinking about issues like congestion control, how to manage retransmissions over a different path, how one party can tell the other about additional addresses (paths) it could use, how to decide when setting up multiple subflows is worth the expense, and so on. The MPTCP designers have done much of that thinking; see [RFC 6824](<https://tools.ietf.org/html/rfc6824>) for the details. 

#### The dreaded middlebox

One set of details merits a closer look, though. The designers of MPTCP are not interested in going through an idle academic exercise; they want to create a solution to real problems that will be deployed on the existing Internet. And that means designing something that will function with the net as it exists now. At one level, that means making things work transparently for TCP-based applications. But there is [an entire section in the RFC](<https://tools.ietf.org/html/rfc6824#section-6>) that is concerned with "middleboxes" and how they can sabotage any attempt to introduce a new protocol. 

Middleboxes are routers that impose some sort of constraint or transformation on network traffic passing through them. Network address translation (NAT) boxes are one example: they hide an entire network behind a translation layer that will change the address and port of a connection on its way through. NAT boxes can also insert data into a stream — adding commands to make FTP work, for example. Some boxes will acknowledge data on its way through, well before it arrives at the real destination, in an attempt to increase pipelining. Some routers will drop packets with unknown options; that behavior made the rollout of the selective acknowledgment (SACK) feature much harder than it needed to be. Firewalls will kill connections with holes in the sequence number stream; they will also, sometimes, transform sequence numbers on the way through. Splitting and coalescing of segments can cause options to be dropped or duplicated. And so on; the list of potential problems is impressive. 

On top of that, anybody trying to introduce an entirely new transport-layer is likely to discover that it will not make it across the Internet at all. Much of the routing infrastructure on the net assumes that TCP and UDP are all there is; anything else has a poor chance of making it through. 

Working around these issues drove the design of MPTCP at all levels. TCP was never designed for multiple subflows; rather than bolting that idea onto the protocol, it might well have been better to start over. One could have incorporated the lessons learned from TCP in all ways — including doing things entirely differently where it made sense. But the resulting protocol would not work on today's Internet, so the designers had no choice but to create a protocol that, to almost every middlebox out there, looks like plain old TCP. 

So every subflow is an independent TCP connection in every respect. Since holes in sequence numbers can cause problems, each subflow has its own sequence and a mapping layer must be added on top. That mapping layer uses relative sequence numbers because some middlebox may have changed those numbers as they passed through. The two sides assign "address identifiers" to the IP addresses of their interfaces and use those identifiers to communicate about those interfaces, since the addresses themselves may be changed by a NAT box in the middle. When one side tells the other about an available interface, it adds an "address identifier" to be used in future messages because a NAT box might change the visible address of that interface. Special checks exist for subflows that corrupt data, insert preemptive acknowledgments, or strip unknown options; such subflows will not be used. And the whole thing is designed to fall back gracefully to ordinary TCP if the interference is too strong to overcome. 

It is all a clever bit of design on the part of the MPTCP developers, but it also highlights an area of concern: the "dumb" Internet with end-to-end transparent routing of data is a thing of the distant past. What we have now is inflexible and somewhat hostile to the deployment of new technologies. The MPTCP developers have been able to work around these limitations, but the effort required was considerable. In the future, we may find that the net is broken in fundamental ways and it simply cannot be fixed; some might say that the difficulties in moving to IPv6 show that this has already happened. 

#### Future directions

The current MPTCP code can be found at [the MPTCP github repository](<https://github.com/multipath-tcp/mptcp>); it adds a good 10,000 lines to the mainline kernel's networking subtree. While it has apparently been the subject of discussions with various networking developers, it has not, yet, been posted for public review or inclusion into the mainline. It does, however, seem to work: the MPTCP developers claim to have implemented [the fastest TCP connection ever](<http://multipath-tcp.org/pmwiki.php?n=Main.50Gbps>) by transmitting at a rate of 51.8Gb/s over six 10Gb links. 

MPTCP is still relatively young, so there is almost certainly quite a bit of work yet to be done before it is ready for mainline merging or production use. There is also some thinking to be done on the application side; it may be possible for MPTCP-aware applications to make better use of the available paths. Projects like this are arguably never finished (we are still refining TCP, after all), but MPTCP does seem to have reached the point where more users may want to start experimenting with it. 

Anybody wanting to play with this code can grab the project's kernel repository and build a custom kernel. For those who are not up to that level of effort, the project offers a number of [other options](<http://multipath-tcp.org/pmwiki.php?n=Users.HowToInstallMPTCP>), including a Debian repository, instructions for running MPTCP on Amazon's EC2, and kernels for a handful of Android-based handsets. Needless to say, the developers are highly interested in hearing bug reports or other testing results.
