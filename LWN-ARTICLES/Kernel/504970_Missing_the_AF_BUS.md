---
title: Missing the AF_BUS
url: https://lwn.net/Articles/504970/
date: "July 3, 2012"
category: "D-Bus; Message passing; Networking-D-Bus"
author: "By Jonathan Corbet July 3, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 3, 2012

The [D-Bus](<http://www.freedesktop.org/wiki/Software/dbus>) interprocess communication mechanism has, over the years, become a standard component of the Linux desktop. For almost as long, developers have been trying to find ways to make D-Bus faster. The latest attempt comes in the form of a kernel patch set adding a new socket address family (called `AF_BUS`) to the networking layer. Significant performance improvements are claimed, but, like previous attempts, this one may have a hard time getting into the mainline kernel. 

D-Bus implements a mechanism by which processes can send messages to each other. Multicast functionality is inherently a part of the protocol; one message can be sent to multiple recipients. D-Bus promises reliable delivery, where "reliable" means that messages arrive in the order in which they were sent and multicast messages will either be delivered to all recipients or, if that is not possible, to none. There is a security model built into the protocol whereby messages can be limited to specific recipients. All of these features are used by contemporary systems, which expect the system to be robust, secure, and with as little latency and overhead as possible. 

The current D-Bus implementation uses Unix-domain sockets and a central routing daemon. It works, but the routing daemon adds context switches, overhead, and latency to each message it handles. The kernel is unable to help get high-priority messages delivered first, so all messages cause wakeups that slow down the processing of the most important ones; see [this message](<https://lwn.net/Articles/504984/>) for a description of how these problems can affect a running system. It has been evident for some time to the developers involved that a better solution must be found. 

There have been a number of attempts in that direction. The previous time this topic came up, it was around [a set of patches](<https://lwn.net/Articles/484203/>) adding multicast capabilities to Unix-domain sockets. This idea was rejected with the claim that the Unix-domain socket code is already too complicated and there was not enough justification to make things worse by adding multicast capabilities. The D-Bus developers were told to simply use IPv4 sockets, which already have multicast support, instead. 

What those developers actually did was to implement [AF_BUS](<https://lwn.net/Articles/504722/>), a new address family designed to meet the needs of D-Bus. It provides the reliable delivery that D-Bus requires; it also has the ability to pass file descriptors and credentials from one process to another. The security mechanism is built in, with the netfilter code (augmented with a new D-Bus message parser) used to control which messages can actually be delivered to any specific process. The end result, it is claimed, is a significant reduction in D-Bus overhead due to reduced system calls; submitter Vincent Sanders claims ""a doubling in throughput and better than halving of latency."" See [the associated documentation](<https://lwn.net/Articles/504988/>) for details on how this address family works. 

A factor-of-two improvement in a component that is widely used in Linux systems would certainly be welcome. The patch set, however, was not; networking maintainer David Miller immediately [stated his intention](<https://lwn.net/Articles/504977/>) to simply ignore the patch set entirely. His objections seem to be that IPv4 sockets are sufficient for the task and that reliable delivery of multicast messages cannot be done, even in the limited manner needed by D-Bus. He expressed doubts that the IPv4 approach had even been tried, and [decreed](<https://lwn.net/Articles/504978/>): ""We are not creating a full address family in the kernel which exists for one, and only one, specific and difficult user."" 

Vincent [responded](<https://lwn.net/Articles/504979/>) that a number of approaches have been tried and found wanting. IPv4 sockets cannot provide the needed delivery guarantees and do not allow for the passing of file descriptors and credentials. It is also important, he said, for D-Bus to be up and running before the networking subsystem has been configured; setting up IP interfaces on a contemporary system often requires communication over D-Bus. There really is no better solution, he said. 

He found support from a few other developers, including [Alan Cox](<https://lwn.net/Articles/504980/>), who pointed out that there is no shortage of interprocess communication systems out there with requirements similar to D-Bus: 

In fact if you look up the stack you'll find a large number of multicast messaging systems which do reliable transport built on top of IP. In fact Red Hat provides a high level messaging cluster service that does exactly this. (as well as dbus which does it on the desktop level) plus a ton of stuff on top of that (JGroups etc) 

Everybody at the application level has been using these 'receiver reliable' multicast services for years (Websphere MQ, TIBCO, RTPGM, OpenPGM, MS-PGM, you name it). There are even accelerators for PGM based protocols in things like Cisco routers and Solarflare can do much of it on the card for 10Gbit. 

He added that latency concerns are paramount on contemporary systems and that one of the best ways of reducing latency is to cut back on context switches and middleman processes. Chris Friesen [added](<https://lwn.net/Articles/504981/>) that his company uses ""an out-of-tree datagram multicast messaging protocol family based on AF_UNIX"" that could almost certainly be replaced by something like `AF_BUS`, were `AF_BUS` to be added to the mainline kernel. 

There have been various other local messaging patch sets posted over the years. So it seems clear that there is a significant level of interest in having this sort of capability built into the Linux kernel. But interest alone is not sufficient justification for the merging of a large patch set; there must also be agreement from the developers who are charged with ensuring that Linux has a top-quality networking stack in the long term. That agreement is not yet there, so there may be a significant amount of multicast interpersonal messaging required before we have multicast interprocess messaging in the kernel.
