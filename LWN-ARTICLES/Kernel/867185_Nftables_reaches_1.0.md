---
title: Nftables reaches 1.0
url: https://lwn.net/Articles/867185/
date: "August 27, 2021"
category: "Networking-Packet filtering; Nftables"
author: "By Jonathan Corbet August 27, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 27, 2021

The Linux kernel is a fast-moving project, but change can still be surprisingly slow to come at times. The [nftables project](<https://nftables.org/projects/nftables/index.html>) to replace the kernel's packet-filtering subsystem has its origins in 2008, but is still not being used by most (or perhaps even many) production firewalls. The transition may be getting closer, though, as highlighted by the release of [nftables 1.0.0](<https://lwn.net/ml/netdev/20210819173626.GA1776@salvia/>) on August 19\. 

The [first public nftables release](<https://lwn.net/Articles/324251/>) was made by Patrick McHardy in early 2009. At that time, the kernel had a capable packet-filtering subsystem in the form of iptables, of course, that was in widespread use, but there were a number of problems driving a change. These include the fact that the kernel had (and still has) more than one packet-filtering mechanism: there is one for IPv4, another for IPv6, yet another for ARP, and so on. Each of those subsystems is mostly independent, with a lot of duplicated code. Beyond that, iptables contains an excessive amount of built-in protocol knowledge and suffers from a difficult API that, among other things, makes it impossible to update a single rule without replacing the entire set. 

The core idea behind nftables was to throw away all of that protocol-aware machinery and replace it with a simple virtual machine that could be programmed from user space. Administrators would still write rules referring to specific packet-header fields and such, but user-space tooling would translate those rules into low-level fetch and compare operations, then load the result into the kernel. That resulted in a smaller packet-filtering engine that was also far more flexible; it also had the potential to perform better. It looked like a win, overall, once the minor problem of transitioning a vast number of users had been overcome. 

Nftables made a bit of a splash when it was launched, but then bogged down and disappeared from view, perhaps because McHardy decided he had [more interesting opportunities to pursue](<https://lwn.net/Articles/721458/>) in courtrooms. In 2013, though, Pablo Neira Ayuso [restarted the project](<https://lwn.net/Articles/564095/>) with the idea of getting the code merged into the mainline as soon as possible. That part succeeded; nftables found its way into the 3.13 kernel release at the beginning of 2014. 

The work since then has been a hard slog of filling in the gaps and making nftables sufficiently appealing that users would want to make the transition. The language used to write filtering rules has gained a long list of features for stateful tracking, address mapping, efficient handling of address intervals and large rule chains, and support for numerous protocols. There was also documentation to write, of course; the [nftables wiki](<https://wiki.nftables.org/wiki-nftables/index.php/Main_Page>) has a lot of information about how it all works. 

There is, of course, one other significant impediment to transitioning away from iptables: the vast number of deployed, working firewalls using the latter. In many cases, rewriting the firewall rules may be the best course of action because many complex filtering setups can be expressed much more efficiently in the new scheme. But, for administrators who just want their painfully developed firewall to keep working, the benefits of nftables may be less appealing than one might expect. The nftables developers have developed a set of scripts to translate iptables firewalls into the nftables equivalent, which should help, but it is still a big jump. 

In some cases, users may eventually make that jump without even noticing, though. Linux distributions have carried support for nftables for some time now, and work is being done [to port tools like Red Hat's firewalld](<https://firewalld.org/2018/07/nftables-backend>) to nftables. In cases like this, users may have never seen the iptables rules in the first place and, with luck, will not notice that the underlying mechanism has changed. 

When will that change happen? It is still somewhat hard to say. The [2018 Netfilter Workshop](<https://ral-arturo.org/2018/06/16/nfws2018.html>) decreed that iptables is ""a legacy tool"" whose days are numbered. Debian switched to nftables by default in the 2019 Debian 10 "buster" release, though Ubuntu didn't follow until the 21.04 release. While almost all distributions ship nftables, many of them have yet to make the switch to use it by default. 

The release of nftables 1.0.0 can be seen as a signal that it is time for the laggards to get more serious about making the switch. While it is hard to imagine iptables support being removed anytime soon, it's rather easier to foresee that enthusiasm for maintaining it will continue to wane. New features will show up in nftables instead, and users will eventually need to migrate over to take advantage of them. It only took 13 years, but this transition finally appears to be heading into its final stage. 

There is, however, one other interesting question. In 2018, the BPF developers [announced bpfilter](<https://lwn.net/Articles/747551/>), a packet-filtering mechanism that runs on the BPF virtual machine. The announcement drew some attention at the time; BPF had (and has) a lot of momentum, and a lot of work has been done to optimize the virtual machine and make it safe to use. Arguably, it makes sense to use that rather than maintain yet another virtual machine just for packet filtering. That would allow the removal of a bunch of code and the focusing of maintenance effort on BPF. 

The bpfilter code was [merged for the 4.18 kernel release](<https://lwn.net/Articles/755919/>); it also brought in a "user-mode blobs" mechanism that was intended to facilitate the translation of firewall rules to the new machine. Since then, however, development on this code has come to a halt; there have been exactly two (trivial) commits to the code in `net/bpfilter` in 2021. The removal of this code was [discussed in June 2020](<https://lwn.net/Articles/822744/>) but it survived at that time. Since then, the cobwebs have only gotten thicker; it seems fair to say that bpfilter is not an active area of development at this point, and that it seems unlikely to displace nftables anytime soon. 

Whether that is the "right" outcome is hard to say. Perhaps the special-purpose virtual machine used by nftables is a better solution to this particular problem than the more general BPF. Or possibly nftables came out on top simply because the developers behind it continued to show up and push the project forward. One of the keys to success in kernel development is simple persistence; that is doubly true for a critical subsystem like packet filtering, where it is more than reassuring to know that the developers are in it for the long haul.
