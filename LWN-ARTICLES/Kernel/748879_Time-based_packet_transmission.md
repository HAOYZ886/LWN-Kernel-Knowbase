---
title: "Time-based packet transmission"
url: https://lwn.net/Articles/748879/
date: "March 8, 2018"
category: Networking; Realtime
author: "By Jonathan Corbet March 8, 2018"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 8, 2018

Normally, when an application sends data over the network, it wants that data to be transmitted as quickly as possible; the kernel's network stack tries to oblige. But there are applications that need their packets to be transmitted within specific time windows. This behavior can be approximated in user space now, but a better solution is in the works in the form of the [time-based packet transmission](<https://lwn.net/Articles/748744/>) patch set. 

There are a number of situations where outgoing data should not necessarily be transmitted immediately. One example would be any sort of isochronous data stream — an audio or video stream, maybe — where each packet of data is relevant at a specific point in time. For such streams, transmitting ahead of time and buffering at the receiving side generally works well enough. But realtime control applications can be less flexible. Commands for factory-floor or automotive systems, for example, should be transmitted within a narrow period of time. Realtime applications can wait until the window opens before queuing data for transmission, of course, but any sort of latency that creeps in (due to high network activity, for example) may then cause the data to be transmitted too late. 

Naturally, the network-standards community has been working on solutions for this particular problem; one of them is called [P802.1Qbv](<http://www.ieee802.org/1/pages/802.1bv.html>). Should that name prove to be a mouthful, there is the more concise alternative: "Standard for Local and Metropolitan Area Networks-Media Access Control (MAC) Bridges and Virtual Bridged Local Area Networks Amendment: Enhancements for Scheduled Traffic". It defines a mechanism for the draining of queues of packets such that each packet is transmitted by its specific deadline. When P802.1Qbv is in use, applications can queue packets whenever they are ready, but those packets will not actually hit the wire until their deadline approaches. 

The patch set implementing time-based transmission on Linux has a few separate components to it. The first is an API addition to allow applications to request this behavior. That is done by setting the new `SO_TXTIME` option with the [`setsockopt()`](<http://man7.org/linux/man-pages/man2/setsockopt.2.html>) system call. Packets intended for timed transmission should be sent with [`sendmsg()`](<http://man7.org/linux/man-pages/man2/send.2.html>), with a control-message header (of type `SCM_TXTIME`) indicating the transmission deadline as a 64-bit nanoseconds value. 

There are a couple of other control-message parameters that can be set with `sendmsg()`. `SCM_DROP_IF_LATE` instructs the network stack to simply drop a packet if, for some reason, it cannot be transmitted by the given deadline. The `SCM_CLOCKID` message can be used to specify which clock should be used for packet timing; the default is `CLOCK_MONOTONIC`. This parameter does not appear to actually be used in the current implementation, though, with one small exception described below. 

These changes to the core network stack allow the specification of time-based behavior, but the core itself does not implement that behavior. That, instead, is an add-on feature. One way to get it is with the `tbs` queuing discipline, which is also part of the patch set. It can be configured to use time-based scheduling on a specific queue, with a couple of additional parameters. Here, too, the clock ID can specified; if the clock ID also appears in individual packets the two must match, or the packets will be dropped. There is also a `delta` parameter to configure how far in advance of the deadline each packet should be sent to the network interface for transmission. This parameter and the deadline for each packet thus define the window in which the packet should hit the wire. 

The delta and the `SCM_DROP_IF_LATE` flag can be used to obtain two distinctly different behaviors. If the flag is set and delta is reasonably large, the semantics are that the packet must be transmitted before the given deadline. Instead, with a small (or zero) delta and with `SCM_DROP_IF_LATE` not set, the behavior is to not transmit the packet until after the given deadline. 

The `tbs` queuing discipline, by itself, is a "best-effort" implementation, since there is still the possibility that packets could be delayed after `tbs` releases them to the interface. The real intent behind P802.1Qbv, however, appears to be implementation in the network adapters themselves. If the adapter is aware of packet deadlines, it can schedule its own transmission activities to ensure that the packets hit the wire at the right time. 

The `tbs` queuing discipline thus supports offloading time-based transmission to the hardware; the patch set includes an implementation for the Intel `igb` Ethernet driver. In a full offload scenario, the delta and clock-ID parameters are not used; instead, all deadlines are assumed to be relative to the clock running within the adapter itself, so the adapter takes full responsibility for packet timing. If those parameters are specified, instead, `tbs` will sort the packets and send them to the interface at the beginning of the transmission window, with the interface still taking responsibility for getting them out before the deadline passes. Since this mode uses both a kernel-based clock and the adapter's own clock, the two must be running in sync or the results will not be as desired. 

The patch set is now in its third revision; the initial version was posted by Richard Cochran but it is now being posted by Jesus Sanchez-Palencia, who has made a number of changes and added the hardware offload capability. There is still some disagreement over how the API should work and, in particular, if the ability to specify different clocks is really needed. Storing a clock ID with each packet makes the network stack's `sk_buff` structure larger, which is something that the networking developers have been resisting strongly for some time now. Working that out is likely to take at least one more revision, so it's not clear if this patch set will be ready by the 4.17 merge window or not.
