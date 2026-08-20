---
title: "Better visibility into packet-dropping decisions"
url: https://lwn.net/Articles/885729/
date: "February 25, 2022"
category: Networking
author: "By Jonathan Corbet February 25, 2022"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 25, 2022

Dropped packets are a fact of life in networking; there can be any number of reasons why a packet may not survive the journey to its destination. Indeed, there are so many ways that a packet can meet its demise that it can be hard for an administrator to tell why packets are being dropped. That, in turn, can make life difficult in times when users are complaining about high packet-loss rates. Starting with 5.17, the kernel is getting some improved instrumentation that should shed some light on why the kernel decides to route packets into the bit bucket. 

This problem is not new, and neither are attempts to address it. The kernel currently contains a "drop_monitor" functionality that was introduced in the 2.6.30 kernel back in 2009. Over the years, it has gained some functionality but has managed to remain thoroughly and diligently undocumented. This feature appears to support a netlink API that can deliver notifications when packets are dropped. Those notifications include an address within the kernel showing where the decision to drop the packet was made, and can optionally include the dropped packets themselves. User-space code can turn the addresses into function names; desperate administrators can then dig through the kernel source to try to figure out what is going on. 

It seems like there should be a better way. As it happens, the beginning of the infrastructure to provide that better way was [contributed to 5.17](<https://git.kernel.org/linus/c504e5c2f964>) by Menglong Dong. The internal kernel function that frees the memory holding a packet is `kfree_skb()`; in 5.17, that function has become: 

```
void kfree_skb_reason(struct sk_buff *skb, enum skb_drop_reason reason);
```

The `reason` argument is new; it is intended to say _why_ the packet passed as `skb` has reached the end of the line. This information is not actually useful to the kernel, but it has been added to the existing `kfree_skb` tracepoint, making it available to any program that connects to that tracepoint. Analysis scripts can quickly print out why packets are being dropped; administrators can also attach BPF programs to, for example, create a histogram of reasons for dropped packets. 

A new version of `kfree_skb()` has also been added; it simply calls `kfree_skb_reason()` with "unspecified" as the reason. 

In 5.17, the use of this infrastructure is relatively limited. There are a few TCP-level drop locations that have been instrumented with the new call, including code that drops packets for being smaller than the TCP header size, not being associated with an existing TCP socket, exhibiting checksum failures, or having been explicitly dropped by an add-on socket filter program. The UDP subsystem has also been enhanced to note those same reasons for dropped packets. 

The situation is set to improve considerably in 5.18; patches already in linux-next add a number of new reasons. These document packets dropped by the netfilter subsystem, that contain IP-header errors, or have been identified as a spoofed packet by the reverse-path filter (rp_filter) mechanism. Administrators will be able to see IP packets that have been dropped due to an unsupported higher-level protocol. Reasons have also been added for UDP packets dropped by the [IPSec XFRM](<https://man7.org/linux/man-pages/man8/ip-xfrm.8.html>) policy or a lack of memory within the kernel. 

There is [yet another set](<https://lwn.net/ml/linux-kernel/20220220070637.162720-1-imagedong@tencent.com/>) of reason annotations that has been accepted, but which has not yet found its way into linux-next; chances are that these will show up in 5.18 as well. They extend the XFRM-policy annotation to TCP, note packets dropped due to missing or incorrect [MD5 hashes](<https://datatracker.ietf.org/doc/html/rfc2385>) (which are evidently still a thing in 2022), as well as those containing invalid TCP flags or sequence numbers outside of the current TCP window. These patches also add new instances of the other reasons noted above; some situations can be detected in multiple places. 

While the above set of reasons may seem long, this work could be seen as having just begun. In current linux-next, there are over 2,700 calls to `kfree_skb()`, compared to 18 to `kfree_skb_reason()`. That suggests that a lot of packets will still be dropped for unspecified reasons. Still, this work represents a useful step forward, one that should make many of the reasons for packet loss more readily available to system administrators. 

The part that remains missing, of course, is the user-space side. The current reason codes are all [defined in `<linux/skbuff.h>`](<https://elixir.bootlin.com/linux/v5.17-rc5/source/include/linux/skbuff.h#L310>), which is not part of the externally available kernel API. Moving them to a separate file under the `uapi` directory would make them more accessible to developers. Also helpful, of course, would be to have some documentation for this mechanism and how to use it (and interpret the results), but even your editor, often cited for naive optimism, will not be holding his breath for that to show up. 

Meanwhile, though, an important piece of the kernel's network functionality is becoming a little more transparent to users. That should make life easier for system administrators who will be able to spend less time trying to figure out why packets aren't making it through the system. Unfortunately, though, this work offers no help for users who are wondering why their packets are disappearing somewhere in the far reaches of the Internet.
