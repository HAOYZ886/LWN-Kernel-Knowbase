---
title: TCP connection repair
url: https://lwn.net/Articles/495304/
date: "May 1, 2012"
category: Checkpointing; Containers; Networking
author: "By Jonathan Corbet May 1, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 1, 2012

Migrating a running container from one physical host to another is a tricky job on a number of levels. Things get even harder if, as is likely, the container has active network connections to processes outside of that container. It is natural to want those connections to follow the container to its new host, preferably without the remote end even noticing that something has changed, but the Linux networking stack was not written with this kind of move in mind. Even so, it appears that transparent relocation of network connections, in the form of Pavel Emelyanov's [TCP connection repair patches](<https://lwn.net/Articles/493983/>), will be supported in the 3.5 kernel. 

The first step in moving a TCP connection is to gather all of the information possible about its current state. Much of that information is available from user space now; by digging around in `/proc` and `/sys`, one can determine the address and port of the remote end, the sizes of the send and receive queues, TCP sequence numbers, and a number of parameters negotiated between the two end points. There are still a few things that user space will need to obtain, though, before it can finish the job; that requires some additional support from the kernel. 

With Pavel's patch, that support is available to suitably privileged processes. To dig into the internals of an active network connection, user space must put the associated socket into a new "repair mode." That is done with the `setsockopt()` system call, using the new `TCP_REPAIR` option. Changing a process's repair mode status requires the `CAP_NET_ADMIN` capability; the socket must also either be closed or in the "established" state. Once the socket is in repair mode, it can be manipulated in a number of ways. 

One of those is to read the contents of the send and receive queues. The send queue contains data that has not yet been successfully transmitted to the remote end; that data needs to move with the connection so it can be transmitted from the new location. The receive queue, instead, contains data received from the remote end that has not yet been consumed by the application being moved; that data, too, should move so it will be waiting on the new host when the application gets around to reading it. Obtaining the contents of these queues is done with a two-step sequence: (1) call `setsockopt(TCP_REPAIR_QUEUE)` with either `TCP_RECV_QUEUE` or `TCP_SEND_QUEUE`, then (2) call `recvmesg()` to read the contents of the selected queue. 

It turns out there is only one other important piece of information that cannot already be obtained from user space: the maximum value of the MSS (maximum segment size) negotiated between the two endpoints at connection setup time. To make this value available, Pavel's patch changes the semantics of the `TCP_MAXSEG` socket option (for `getsockopt()`) when the connection is in repair mode: it returns the maximal "clamp" MSS value rather than the currently active value. 

Finally, if a connection is closed while it is in the repair mode, it is simply deleted with no notification to the remote end. No FIN or RST packets will be sent, so the remote side will have no idea that things have changed. 

Then there is the matter of establishing the connection on the new host. That is done by creating a new socket and putting it immediately into the repair mode. The socket can then be bound to the proper port number; a number of the usual checks for port numbers are suspended when the socket is in repair mode. The `TCP_REPAIR_QUEUE` `setsockopt()` call comes into play again, but this time `sendmsg()` is used to restore the contents of the send and receive queues. 

Another important task is to restore the send and receive sequence numbers. These numbers are normally generated randomly when the connection is established, but that cannot be done when a connection is being moved. These numbers can be set with yet another call to `setsockopt()`, this time with the `TCP_QUEUE_SEQ` option. This operation applies to whichever queue was previously selected with `TCP_REPAIR_QUEUE`, so the refilling of a queue's content and the setting of its sequence number are best done at the same time. 

A few negotiated parameters also need to be restored so that the two ends will remain in agreement with each other; these include the MSS clamp described above, along with the active maximum segment size, the window size, and whether the selective acknowledgment and timestamp features can be used. One last `setsockopt()` option, `TCP_REPAIR_OPTIONS`, has been added to make it possible to set these parameters from user space. 

Once the socket has been restored to a state approximating that which existed on the old host, it's time to put it into operation. When `connect()` is called on a socket in repair mode, much of the current setup and negotiation code is shorted out; instead, the connection goes directly to the "established" state without any communication from the remote end. As a final step, when the socket is taken out of the repair mode, a window probe is sent to restart traffic between the two ends; at that point, the socket can resume normal operation on the new host. 

These patches have been through a few revisions over a number of months; with version 4, networking maintainer David Miller [accepted](<https://lwn.net/Articles/495318/>) them into net-next. From there, those changes will almost certainly hit the mainline during the 3.5 merge window. The TCP connection repair patches do not represent a complete solution to the problem of checkpointing and restoring containers, but they are an important step in that direction.
