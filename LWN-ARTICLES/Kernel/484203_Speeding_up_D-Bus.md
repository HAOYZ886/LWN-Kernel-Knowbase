---
title: "Speeding up D-Bus"
url: https://lwn.net/Articles/484203/
date: "February 29, 2012"
category: "D-Bus; Message passing; Networking-D-Bus"
author: "By Jake Edge February 29, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
February 29, 2012

The [D-Bus](<http://www.freedesktop.org/wiki/Software/dbus>) interprocess communication (IPC) mechanism is used extensively by Linux desktop environments and applications, but it suffers from less-than-optimal performance. While that [problem](<http://alban-apinc.blogspot.com/2011/12/d-bus-in-kernel-faster.html>) may not be so noticeable on desktop-class systems, it can be a real issue for smaller and embedded devices. Over the years there have been a number of attempts to add functionality to the kernel to increase D-Bus performance, but none have passed muster. A recent [proposal](<https://lwn.net/Articles/482523/>) to add multicast functionality to Unix sockets is another attempt at solving that problem. 

D-Bus currently relies on a daemon process to authenticate processes and deliver messages that it receives over Unix sockets. Part of the performance problem is caused by the user-space daemon, which means that messages need two trips through the kernel on their way to the destination (once on the way to the daemon and another on the way to the receiver). It also requires waking up the daemon and an "extra" transition to and from kernel mode. Putting D-Bus message handling directly into the kernel would eliminate the need to involve the daemon at all. That would eliminate one of the transitions and one of the copies, which would improve performance. 

If all of the D-Bus messages were simply between pairs of processes, Unix sockets could potentially be used directly between them. But there is more to D-Bus than that. Processes can register for certain kinds of events they wish to receive (things like USB devices being attached, a new song playing, or battery status changes for example), so a single message may need to be multicast to multiple receivers. That is part of what the daemon mediates. 

Earlier efforts to [add an `AF_DBUS` socket type](<https://lwn.net/Articles/405346/>) (and associated kdbus module) to handle D-Bus messages in the kernel weren't successful because kernel hackers were not willing to add the complexity of D-Bus routing. The most recent proposal was posted by Javier Martinez Canillas based on work from Alban Créquy, who also proposed the `AF_DBUS` feature. It adds multicasting support to Unix (i.e. `AF_UNIX`) sockets, instead, while using packet filters so that receivers only get the messages they are interested in. That way, the routing is strictly handled via multicast plus existing kernel infrastructure. 

As described in Rodrigo Moya's [blog posting](<http://blogs.gnome.org/rodrigo/2012/02/27/d-bus-optimizations/>), there are a number of reasons that a D-Bus optimization can't use the existing mechanisms in the kernel. Netlink sockets would seem to be one plausible alternative, and there is support for multicasting, but D-Bus requires fully reliable delivery even if the receiver's queue is full. In that case, netlink sockets just drop packets, while D-Bus needs the sender to block until the receiver processes some of its messages. In addition, netlink sockets do not guarantee the ordering of multicast messages that D-Bus requires. 

Another option would be to use UDP multicast, but Moya (and Canillas) seem skeptical that it will perform as well as Unix socket multicast. There is also a problem for devices that do not have a network card, because the `lo` loopback network device does not support multicast. Moya also notes that a UDP-based solution suffers from the same packet loss and ordering guarantee problems that come with netlink sockets. 

So, that left Créquy and others at Collabora (including Moya, Canillas, and others) to try a different approach. Créquy [outlines](<http://alban-apinc.blogspot.com/2011/12/introducing-multicast-unix-sockets.html>) the multicast approach on his blog. Essentially, both `SOCK_DGRAM` and `SOCK_SEQPACKET` socket types can create and join multicast groups which will then forward all traffic to each member of the group. Datagram multicast allows any process that knows the group address to join, while seqpacket multicast (which is connection-oriented like a `SOCK_STREAM` but enforces message boundaries) allows the group creator to decide whether to allow a particular group member at `accept()` time. 

As Moya described, a client would still connect to the D-Bus daemon for authentication, and would then be added to the seqpacket multicast group for the bus. The daemon would also attach a packet filter that would eliminate any of the messages that the client should not receive. One of the patches in the set implements the ability for the daemon to attach a filter to the remote endpoint, so that it would be in control of which messages a client can see. 

The idea is interesting, but so far comments on the netdev mailing list have been light. Kernel network maintainer David Miller is [skeptical](<https://lwn.net/Articles/484350/>) that the proposal is better than having the daemon just use UDP: 

My first impression is that I'm amazed at how much complicated new code you have to add to support groups of receivers of AF_UNIX messages. 

I can't see how this is better than doing multicast over ipv4 using UDP or something like that, code which we have already and has been tested for decades. 

Cannilas [responded](<https://lwn.net/Articles/484448/>) by listing some of the reasons that UDP multicast would not serve their purposes, but admitted that no performance numbers had yet been gathered. Miller said that he will await those numbers before reviewing the proposal further, [noting](<https://lwn.net/Articles/484355/>): ""In many cases TCP/UDP over loopback is actually faster than AF_UNIX."". 

Even if UDP has the needed performance, some solution would need to be found for the packet loss and ordering issues. Blocking senders due to inattentive receivers may be a hard sell, however, as it seems like it could lead to denial of service problems, no matter which socket type is used. But it is clear that there is a lot of interest in better D-Bus performance. In fact, it goes well beyond just D-Bus as ["fast" IPC mechanisms](<https://lwn.net/Articles/466304/>) are regularly proposed for the kernel. It's unclear whether multicasting for Unix sockets is suitable for that, but finding a way to speed up D-Bus (and perhaps other IPC-using programs) is definitely on some folks' radar.
