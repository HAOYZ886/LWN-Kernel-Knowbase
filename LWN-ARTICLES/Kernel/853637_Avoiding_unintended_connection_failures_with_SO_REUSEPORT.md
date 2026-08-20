---
title: Avoiding unintended connection failures with SO_REUSEPORT
url: https://lwn.net/Articles/853637/
date: "April 23, 2021"
category: "Networking-SO REUSEPORT"
author: "By Jonathan Corbet April 23, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 23, 2021

Many of us think that we operate busy web servers; LWN's server, for example, sweats hard when keeping up with the comment stream that accompanies any article mentioning the Rust programming language. But some organizations run truly _busy_ servers and have to take some extraordinary measures to keep up with levels of traffic that even language advocates cannot create. The `SO_REUSEPORT` socket option is one of many features that have been added to the network stack to help these use cases. `SO_REUSEPORT` suffers from an implementation problem that can cause connections to fail, though. Kuniyuki Iwashima has posted [a patch set](<https://lwn.net/ml/linux-kernel/20210420154140.80034-1-kuniyu@amazon.co.jp/>) addressing this problem, but there is some doubt as to whether it takes the right approach. 

In normal usage, only one process is allowed to bind to any given TCP port to accept incoming connections. On busy systems, that process can become a bottleneck, even if all it does is pass accepted connections off to other processes for handling. The `SO_REUSEPORT` socket option, which was [added to the 3.9 kernel](<https://lwn.net/Articles/542629/>) in 2013, was meant to address that bottleneck. This option allows multiple processes to accept connections on the same port; whenever a connection request comes in, the kernel will pick one of the listening processes as the recipient. Systems using `SO_REUSEPORT` can dispense with the dispatcher process, improving scalability overall. 

`SO_REUSEPORT` does its work when the initial `SYN` packet (the connection request) is received; at that time, a provisional new socket is created and assigned to one of the listening processes. The new connection will first wait for the handshake to complete, after which it will sit in a queue until the selected process calls [`accept()`](<https://man7.org/linux/man-pages/man2/accept.2.html>) to accept the connection and begin the session. On busy servers, there may be a fair number of connections awaiting acceptance; the maximum length of that queue is specified with the [`listen()`](<https://man7.org/linux/man-pages/man2/listen.2.html>) system call. 

#### When SO_REUSEPORT misbehaves

Most of the time, `SO_REUSEPORT` works just as intended, but that can change when one of the listening processes exits. If a process quits with open network connections, those connections will be closed — not a surprising result. But the kernel will also "close" (by resetting) any incoming connections that are still in the accept queue. In the absence of `SO_REUSEPORT` this behavior makes sense; if the (single) listening process goes away, there is no longer anybody who can accept those connections. 

If `SO_REUSEPORT` is being used, and if there are multiple listening processes, incoming connections do not necessarily have to be closed in this way. There are, after all, other processes running that would happily handle those connections. But once the kernel has committed an incoming connection to a specific process, it will not change its mind later; either that connection will be accepted by the chosen process, or it will be closed. 

There are a number of reasons why a listening process might exit on a busy system. Perhaps it simply crashed. But, more likely, the server is being restarted to effect a configuration change or to switch to a new certificate. Such restarts can be phased across a pool of server processes so that they don't all exit at once; that should allow incoming connections to be handled without any apparent interruption of service. But when the above-described behavior comes into the picture, users can be turned away, which tends to have an unpleasant effect on their mood. The depressive effect on the operator of the site, who may have just lost the opportunity to learn that the would-be user is in the market for a new pair of socks, can be even worse. 

There are ways around this problem, such as using a BPF program to steer incoming connections away from a server process that is about to exit, then being sure that it drains any queued connections before it bows out. But Iwashima makes the point that there is a better way: when a process exits, just take all of the queued incoming connections and reassign them to a different process for handling. After all, there is no state yet associated with an unaccepted request; one process can handle it just as well as another, and moving it will avoid causing the request to fail. 

#### Migrating the accept queue

Getting there requires an eleven-part patch set, though. The first step is to add a new sysctl knob (`net.ipv4.tcp_migrate_req`; there does not appear to be an IPv6 version) controlling whether incoming connections should be moved to a new listener if the one they were assigned to exits. By default, this new behavior is disabled to avoid interfering with deployments where other arrangements have been made. 

Actually migrating incoming connections away from an exiting server process is a bit more complicated than one might think because the "accept queue" is a bit more complicated than has been discussed so far. Remember that TCP connections go through a three-way handshake before being established: the connection is initiated with a `SYN` packet, the server side responds with `SYN+ACK`, and the initiator completes the connection with an `ACK` packet. That entire process must complete before an incoming connection can be given to a server process via `accept()`. 

Fully established connections — those that have completed the three-way handshake but which have not yet been accepted — are relatively easy to move to a new server process; they are just shunted from one queue to another. Connections that are still in the handshake are more complicated, though. They can only be moved at specific points during the sequence; when the handshake completes being the most obvious such point. It is also possible to move a connection when the `SYN+ACK` is retransmitted, should that be necessary. Either way, the remains of the old server process's socket structure must stay around for long enough to finish the handshake; that adds a certain amount of complexity. 

One remaining question is: how is the new recipient for the connection chosen? Normally, the kernel will use the same algorithm it uses to pick a recipient in the first place, which is essentially a round-robin approach. But there will surely be users who know better and who want to be able to redirect these connections more explicitly. For those users there is, inevitably, a new BPF program type (`BPF_PROG_TYPE_SK_REUSEPORT`) that can be used to make decisions on where to reroute these connections. 

#### Unacceptable?

As of this writing, the only comment on Iwashima's patch set [comes from Eric Dumazet](<https://lwn.net/ml/linux-kernel/67fb2df2-3703-4ce9-62d0-ba15435c5a0b@gmail.com/>), who questioned whether it is the right approach. Since `SO_REUSEPORT` was added, he said, the TCP accept code has been reworked to run locklessly, which should address much of the scalability problem that `SO_REUSEPORT` was added to mitigate in the first place. Thus, he said, it might be better for applications to go back to a single-listener mode, perhaps helped by a new form of `accept()` that would allow incoming connections to be quickly directed to server processes. 

That, of course, is a rather different development direction than Iwashima has taken so far and is thus unlikely to be welcome news. One could argue that, while a new `accept()` call might be a more pleasing solution to the overall problem, there should still be a place for a patch series making an existing kernel feature work without occasionally killing incoming connections. So it's not clear how this will play out; stay tuned.
