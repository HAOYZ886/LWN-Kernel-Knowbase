---
title: ICMP sockets
url: https://lwn.net/Articles/420799/
date: "December 22, 2010"
category: Networking
author: "By Jonathan Corbet December 22, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 22, 2010

The Openwall Linux developers have an interesting problem: they have managed to create a distribution which is entirely free of setuid-root binaries, with one exception: `ping` still needs to be setuid root to be able to send ICMP echo packets. That seems a little untidy, so the project put together [a patch](<https://lwn.net/Articles/420800/>) which allows `ping` to run as an unprivileged user. It implements a new type of socket protocol (`IPPROTO_ICMP`) which, despite its name, is not usable for ICMP communications in general. The only type of message which is allowed through is `ICMP_ECHO` (and the associated replies). 

Interestingly, this patch has been trimmed down from the version which is applied to Openwall kernels. In the full version, the ability to create ICMP sockets is restricted to a specific group, which can be set by way of a sysctl knob. The `ping` binary is then installed setgid. In this way, full access to ICMP sockets is not given to unprivileged users, while `ping` only gets enough privilege to create such sockets. The group check was removed from the posted patch to make acceptance easier, but it seems likely to be added back before the next posting. 

For more information about the thinking behind this design, see [this message from Solar Designer](<https://lwn.net/Articles/420801/>).
