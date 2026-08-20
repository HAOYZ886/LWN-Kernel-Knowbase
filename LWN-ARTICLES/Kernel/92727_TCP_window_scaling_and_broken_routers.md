---
title: TCP window scaling and broken routers
url: https://lwn.net/Articles/92727/
date: "July 7, 2004"
category: "Networking-Window scaling; TCP; Window scaling"
author: 
---

Every TCP packet includes, in the header, a "window" field which specifies how much data the system which sent the packet is willing and able to receive from the other end. The window is the flow control mechanism used by TCP; it controls the maximum amount of data which can be "in flight" between two communicating systems and keeps one side from overwhelming the other with data. 

In the early days of TCP, windows tended to be relatively small. The computers of that age did not have huge amounts of memory to dedicate toward buffering network data, and the available networking technology was not fast enough to make use of a larger window in any case. Modern network interfaces can handle larger packets and keep more of them in flight at any given time; they will perform better with a larger window. Some kinds of high-speed long-haul links can have very high bandwidth, but also high latency. Keeping that sort of pipe filled can require a very large window; if a sending system cannot have a large number of packets in transit at any given time, it will not be able to make use of the bandwidth available. For these reasons, good performance can often require very large windows. 

The TCP window field, however, is only 16 bits wide, allowing for a maximum window size of 64KB. The TCP designers must have thought that nobody would ever need a larger window than that. But 64KB is not even close to what is needed in many situations today. The solution to this problem is called "window scaling." It is not new; window scaling was codified in [RFC 1323](<http://www.faqs.org/rfcs/rfc1323.html>) back in 1992\. It is also not complicated: a system wanting to use window scaling sets a TCP option containing an eight-bit scale factor. All window values used by that system thereafter should be left-shifted by that scale factor; a window scale of zero, thus, implies no scaling at all, while a scale factor of five implies that window sizes should be shifted five bits, or multiplied by 32. With this scheme, a 128KB window could be expressed by setting the scale factor to five and putting 4096 in the window field. 

To keep from breaking TCP on systems which do not understand window scaling, the TCP option can only be provided in the initial SYN packet which initiates the connection, and scaling can only be used if the SYN+ACK packet sent in response also contains that option. The scale factor is thus set as part of the setup handshake, and cannot be changed thereafter. 

The details are still being figured out, but it would appear that some routers on the net are rewriting the window scale TCP option on SYN packets as they pass through. In particular, they seem to be setting the scale factor to zero, but leaving the option in place. The receiving side sees the option, and responds with a window scale factor of its own. At this point, the initiating system believes that its scale factor has been accepted, and scales its windows accordingly. The other end, however, believes that the scale factor is zero. The result is a misunderstanding over the real size of the receive window, with the system behind the firewall believing it to be much smaller than it really is. If the expected scale factor (and thus the discrepancy) is large, the result is, at best, very slow communication. In many cases, the small window can cause no packets to be transmitted at all, breaking TCP between the two affected systems entirely. 

In the 2.6.7 kernel, the default scale factor is zero; in Linus's BitKeeper tree and the 2.6.7-mm kernels, instead, it has been increased to seven. This change has brought the broken router behavior to light; suddenly people running current kernels are finding that they cannot talk to a number of systems out there. One of the higher-profile affected sites is [packages.gentoo.org](<http://packages.gentoo.org/>). Gentoo users are, unsurprisingly, not pleased. 

As a way of making things work, Stephen Hemminger has proposed [a patch](<https://lwn.net/Articles/92732/>) which adds a calculation to select the smallest scale factor which covers the largest possible window size. The result on most systems is that the scale factor gets set to two. This factor will still be corrupted by broken routers, but the resulting window size (¼ of what it should be) is still large enough to allow communication to happen. 

The patch makes networking with systems behind broken routers work again, but it has been [rejected](<https://lwn.net/Articles/92733/>) anyway. The networking maintainers (and David Miller in particular) believe that the patch simply papers over a problem, and that adding hacks to the Linux network stack to accommodate broken routers is a mistake. If, instead, the situation is left as it is, pressure on the router manufacturers should get the problem fixed relatively quickly. It has been a few years, now, that Linux has a strong enough presence in the networking world that it can get away with taking this sort of position. 

In the mean time, anybody running a current kernel who is having trouble connecting to a needed site can work around the problem with a command like: 

```
echo 0 > /proc/sys/net/ipv4/tcp_default_win_scale
```

or by adding a line like: 

```
net.ipv4.tcp_default_win_scale = 0
```

to `/etc/sysctl.conf`.
