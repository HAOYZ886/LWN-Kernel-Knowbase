---
title: A report from the networking miniconference
url: https://lwn.net/Articles/609571/
date: "August 27, 2014"
category: "BPF; Networking-Networking summits"
author: "By Jonathan Corbet August 27, 2014 Kernel Summit 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 27, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

The second day of the 2014 Kernel Summit included a miniconference for networking subsystem developers. Your editor was unable to attend, but he did get to hear Dave Miller's rapid-fire summary of the topics discussed there. The following report has no hope of being complete — taking notes that quickly is difficult — but, with luck, it covers the most important points. 

Dave started with a couple of quick topics, the first of which was the Stream Control Transmission Protocol (SCTP). In general, he said, the networking layer has a lot of highly abstracted code that is shared between protocol implementations. It has always been hard for SCTP to participate in that sharing, though, due to its concept of "associations." The result has been a lot of code duplication in the SCTP subsystem. Now, it seems, there is a new effort afoot to rework the SCTP implementation and unify the code (to a greater extent) with the rest of the networking subsystem. 

One longstanding suboptimal area in the networking code has been the large hash tables allocated for protocols like TCP at boot time. These tables take a lot of memory; they do not necessarily have to be as big as they are, but there is no way to know what the proper size is when the system is coming up. Now, [![\[Dave Miller\]](https://static.lwn.net/images/conf/2014/ks/DaveMiller-sm.jpg)](<https://lwn.net/Articles/609593/>) though, the networking layer has resizeable hash tables protected by the read-copy-update (RCU) mechanism. These tables can be reallocated as necessary, so there is no longer a need to keep large hash tables throughout the life of the system. 

The [extended Berkeley Packet Filter](<https://lwn.net/Articles/603983/>) (eBPF) work, Dave noted, remains somewhat controversial. The biggest problem seems to be that eBPF developer Alexei Starovoitov has a great deal of energy and reviewers are having a hard time keeping up. So, Dave said, he is going to start pushing back a bit on these patches to get Alexei to slow things down. 

There are concerns, Dave said, about the proposal to add the ability to dereference general pointers to eBPF. The possibility of adding backward branches to the eBPF virtual machine is also worrying to some. Nobody disagrees with Alexei's main goal: the creation of a generic virtual machine that is useful throughout the kernel. But it is important not to lose the protected execution environment that eBPF has always provided; it would not be good if eBPF were to become a source of security holes in the kernel. So there will need to be more restrictive rules about pointer access and a lot more checking, he said. 

Ted Ts'o suggested that the SystemTap developers should have a look at eBPF, as it might make a good replacement for the specially-created kernel modules that are loaded now. But James Bottomley responded that SystemTap needs a thoroughly general execution engine — with wide-ranging access to the kernel — which is something that eBPF is explicitly not trying to be. 

Dave then reported on Pablo Neira Ayuso's report on the [Netfilter workshop](<http://workshop.netfilter.org/2014/>) recently held in France. There has been a lot of work put into the removal of the central lock in the connection-tracking code, making that code quite a bit more efficient. There is also, it seems, a determined effort under way to figure out what it will take to run interfaces at the full hardware speed when the traffic is made up of small packets — an area where the Linux network stack falls behind a bit. 

There is interest in Intel's [Data Plane Development Kit](<http://dpdk.org/>) (DPDK), which is a mechanism that pushes packet handling out to user space. It produces good numbers on benchmarks, Dave said, but, in his opinion, there is always going to be some way to get similar performance with in-kernel code. He mentioned [receive polling](<https://lwn.net/Articles/551284/>) as an example: it gives the desired performance, but still keeps the full Linux network stack available. 

Naturally, there was a discussion of [nftables](<https://lwn.net/Articles/564095/>), the in-kernel virtual machine intended to eventually replace iptables. There has been a lot of work done on the iptables compatibility layer, a command-line interface that makes it possible for administrators to run their existing firewall scripts unchanged under nftables. But that does not mean that nftables will be replacing iptables anytime soon; the two are not compatible at the kernel interface level, so iptables will have to stay around for a long time. There was "a brawl" at the workshop about possibly replacing the nftables virtual machine with eBPF, but there is one major show-stopper in any such plan: nftables allows partial replacement of a firewall configuration, while eBPF, in its current form, would not. 

From there, Dave shifted to encapsulation offloading. Whenever you start encapsulating packets and tunneling them through other transports, you have to worry about issues like where the checksumming happens and how flow distribution is managed. This will become a bigger issue, he said, because UDP encapsulation is going to become ubiquitous; just about every chip out there can checksum UDP packets, so support is easy. But steering the various flows is less so. The networking developers want to avoid the use of deep packet inspection for clean handling of encapsulated flows; to that end, they have come up with a trick using source port numbers to identify flows and steer them accordingly. Other tricks manage the checksumming at various layers; one, called "remote checksum offload," limits checksumming to the outer packet, with inner checksumming done at the receiver. 

Of general interest to the network stack is a whole is the concept of send batching. The network driver interface is designed around sending a single packet at a time; there is no way for a driver to know if there are more packets coming immediately afterward — which there often are. If the driver knew more traffic was coming, it could defer starting the hardware, cutting transmit overhead significantly. The plan is to add a new "transmit flush" operation; if a driver provides that function, it will not start the hardware immediately on receipt of a packet to transmit. Instead, that "kick" will be deferred until the flush function is called. Some tweaking may be called for; deferring hardware startup could cause the wire to go idle, which is not desirable. But that seems to be a solvable problem. 

#### Wireless networking

There was, Dave said, a "prisoner exchange," bringing in some developers from the wireless networking summit. Among the topics discussed was ARP proxying in access points to save power. Access points typically already know the MAC addresses of the systems they talk to; they should be able to answer ARP requests and avoid waking the destination system. It was agreed that this task should be handled in the bridging code, which already had related duties. 

A bigger issue is network function offloading, where bridging chips can manage the forwarding database and take the processor out of the loop entirely. It is a nice feature, but there is one problem: it's all managed either via binary-only drivers or vendor-specific user-space code. OpenWRT, evidently, is "having fits" over these drivers. To try to address this problem, some work is being done to extend the netlink interface to cover some of these functions; then, hopefully, vendors can at least be convinced to work with the standard tools. There is a QEMU-based device being developed to test this code with. 

[![\[John Linville\]](https://static.lwn.net/images/conf/2014/ks/JohnLinville-sm.jpg)](<https://lwn.net/Articles/609594/>) Wireless maintainer John Linville got up briefly to discuss a few issues from the wireless summit. One problem the wireless developers are facing is that Android is still using the "wireless extensions" ABI, which has been deprecated for many years. It seems that it is easy to add vendor-specific operations to wireless extensions, so vendors are doing that. In response, the wireless developers have been adding some options to the current interface to give it some more flexibility. But that work has not immediately translated into vendors switching over. The current plan is to "talk to Google" and try to get it to encourage vendors to move away from the wireless extensions. 

There has been some work to get a firmware dump tool in place. After some discussion, the developers came up with an option using sysfs to get the relevant data. 

Finally, John let it be known that he is getting a little tired of being the wireless maintainer, but he has not yet been able to find a good candidate for a replacement. There are a lot of talented developers working on the wireless stack, he said, but most of those work for hardware vendors. It seems that these vendors are, as a general rule, unenthusiastic about having their developers working to support drivers for other vendors' hardware. So a new wireless maintainer almost certainly needs to work for a hardware-neutral organization — a distributor, for example. If there are any such people out there, John would like to hear from them. 

This session covered a number of other topics. For example, Bluetooth maintainer Marcel Holtmann gave a high-speed update on that subsystem that was far beyond your editor's fingers' ability to follow. Suffice to say that the 3.17 kernel will include Bluetooth 4.1 support. The conclusion that results from all this, clearly, is that there is still a lot going on in the networking subsystem.
