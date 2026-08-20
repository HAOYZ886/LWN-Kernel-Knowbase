---
title: How not to get a protocol implementation merged
url: https://lwn.net/Articles/422649/
date: "January 12, 2011"
category: "Networking-Protocols; UDPCP"
author: "By Jonathan Corbet January 12, 2011"
---

By **Jonathan Corbet**  
January 12, 2011

The "Open Base Station Architecture Initiative" is a consortium of companies which are trying to create an open market for cellular base station hardware. One of the things this initiative has defined is the UDPCP protocol - a UDP-based protocol used for communications between base stations. UDPCP offers reliable transfer, multicast, and more. The Linux kernel does not currently support UDPCP, but Stefani Seibold has posted [a patch](<https://lwn.net/Articles/422473/>) which would add that support to the kernel's network stack. 

There have been a number of comments about this code, but one [observation](<https://lwn.net/Articles/422651/>) by Eric Dumazet is noteworthy: the posted implementation only works with IPv4. The networking developers have made it clear that they are uninterested in accepting an IPv4-only implementation in 2011; IPv6 support is required for any new code. 

Stefani [responded](<https://lwn.net/Articles/422652/>) that no base stations currently provided IPv6 functionality and no customers were interested, so there was no point in adding that support at this time. The answer didn't change, though; the networking developers have no interest in merging code which is guaranteed to need fixing in the near future. Stefani has [described](<https://lwn.net/Articles/422653/>) this requirement as ""dogmatic"", but she also seems to have realized that it's not going to go away. So UDPCP stays out of the mainline for now, but we will, hopefully, eventually see a reworked version with support for IPv6.
