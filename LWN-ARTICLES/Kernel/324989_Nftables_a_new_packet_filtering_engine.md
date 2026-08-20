---
title: "Nftables: a new packet filtering engine"
url: https://lwn.net/Articles/324989/
date: "March 24, 2009"
category: "Networking-Packet filtering; Nftables"
author: "By Jonathan Corbet March 24, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 24, 2009

Packet filtering and firewalling has a long history in Linux. The first filtering mechanism, called "ipfwadm," was released in 1995 for the 1.2.1 kernel. This code was used until the 2.2.0 stable release (January, 1999), when the new "ipchains" module took over. While ipchains was useful, it only lasted until 2.4.0 (January, 2001), when it, too, was replaced by iptables/netfilter, which remains in the kernel now. If netfilter maintainer Patrick McHardy has his way, though, iptables, too, will be gone in the future, replaced by yet another mechanism called "nftables." This article will give an overview of how nftables works, followed by a discussion of the motivations behind this change. 

The [first public nftables release](<http://lwn.net/Articles/324251/>) came out on March 18\. This code has been in the works for a while, though, and the ideas were discussed at the [2008 Netfilter Workshop](<http://workshop.netfilter.org/2008/>). So nftables is not quite as new as it might seem. 

The current iptables code has a lot of protocol awareness built into it. There is, for example, a module dedicated to extracting port numbers from UDP packets which is different from the module concerned with TCP packets. The nftables implementation is entirely different; there is no protocol knowledge built into it at all. Instead, nftables is implemented as a simple virtual machine which interprets code loaded from user space. So nftables has no operation which says anything like "compare the IP destination address to 196.168.0.1"; instead, it would execute code which looks like: 

```
payload load 4 offset network header + 16 => reg 1
        compare reg 1 192.168.0.1
```

(Patrick presents the code in mnemonic form, and your editor will do the same; the actual code loaded into the kernel uses opcodes instead). The first line loads four bytes from the packet, located 16 bytes past the beginning of the network reader, into register 1\. The second line then compares that register against the given network address. 

The language can do a lot more than just comparing addresses, of course. There is, for example, a set lookup feature. Consider the following: 

```
payload load 4 offset network header + 16 => reg 1
        	set lookup reg 1 load result in verdict register
          		{ "192.168.0.1" : jump chain1,
                      "192.168.0.2" : drop,
            	  "192.168.0.3" : jump chain2 }
```

This code will cause packets aimed at 192.168.0.2 to be dropped; for the other two listed addresses, control will be sent to specific rule chains. This set feature allows for multi-branch rules in a way which cannot be done with the current iptables implementation (though the [ipset](<http://ipset.netfilter.org/>) mechanism helps in that regard). The above code also introduces the "verdict register," which records an action to be performed on a packet. In nftables, more than one verdict can be rendered on a packet; it is possible to add a packet to a specific counter, log it, and drop it all in a single chain without the need (as seen in iptables) to repeat tests. 

There are a number of other capabilities built into the nftables virtual machine. There's a set of operations for communicating with the connection-tracking mechanism, allowing connection information to be used in deciding the fate of specific packets. Other operators deal with various bits of packet metadata known to the networking subsystem; these include the length, the protocol type, security mark information, and more. Operators exist for logging packets and incrementing counters. There's also a full set of comparison operations, of course. 

Network administrators are unlikely to be impressed by the idea of programming a low-level virtual machine for their future firewalling needs. The good news is that there will be no need for them to do so. Instead, they'll write higher-level rules which will then be compiled into virtual machine code before being loaded into the kernel. The `nftables` utility does this work, implementing a human-readable language encapsulating most of the needed information about how packets are put together. So, if we look back to the first test described above: 

```
payload load 4 offset network header + 16 => reg 1
        compare reg 1 192.168.0.1
```

The administrator would simply write "`ip daddr 192.168.0.1`" and let `nftables` turn that into the above code. A full (if simple) rule looks something like this: 

```
rule add ip filter output ip daddr 192.168.0.1 counter
```

This rule will count packets sent to 192.168.0.1. 

The new nftables API is based on netlink, naturally. Unlike the current iptables API, it has the ability to modify individual rules without the need to reload the entire configuration. There is also a decompilation facility built into `nftables` that allows the recreation of human-readable rules from the current in-kernel configuration. 

[PULL QUOTE:  This could be a disruptive and expensive transition; the kernel development community will want to see some very good reasons for inflicting this pain on its users.  END QUOTE] All told, it looks like a nicely-designed packet filtering mechanism, but the merging of nftables is likely to be controversial. The iptables mechanism works well, and is widely used; replacing it with code which breaks the user-space API _and_ breaks all existing iptables configurations is guaranteed to raise some eyebrows. This could be a disruptive and expensive transition, even if, as seems necessary, the developers commit to maintaining both iptables and nftables in the mainline for an extended period of time. The kernel development community will want to see some very good reasons for inflicting this pain on its users. 

There are some good reasons, but one should start by noting that it should be possible to create a tool which reads current iptables configurations and converts them to the nftables language - or even directly to kernel virtual machine code. Patrick seems to expect to create such a tool One Of These Days, but it does not exist at this time. 

Some of the reasons for replacing iptables have already been hinted at above. The protocol knowledge built into the iptables code has turned out to be a problem over time; there is a lot of duplicated code doing the same thing (extracting port numbers, say) for different protocols. Even worse, the capabilities and syntax tend to vary from one protocol to the next. By moving all of that knowledge out to user space, nftables greatly simplifies the in-kernel code and allows for much more consistent treatment of all protocols. 

There are a lot of optimization possibilities built into the new system. Some expensive operations (incrementing counters, for example) can be skipped unless the user really needs them. Features like set lookups and range mapping can collapse a whole set of iptables rules into a single nftables operation. Since filtering rules are now compiled, there is also potential for the compiler to optimize the rules further. Traditional firewall configurations tend to perform the same tests repeatedly; a smart nftables compiler could eliminate much of that duplicated work. Unsurprisingly, this optimization remains on the "to do" list for now, but the fact that all of this work is done in user space will make it easy to add such features in the future. 

The nftables tool will also be able to perform a higher level of validation on the rules it is given, and it will be able to provide more useful diagnostics than can be had from the iptables code. 

But, arguably, the most important motivation is the ability to dump the current ABI. The iptables ABI has become an increasing impediment to development over time. It includes protocol-specific fields which has made it hard to extend; that is part of why there are actually three copies of the iptables code in the kernel. When developers wanted to implement arptables and ebtables, they essentially had to copy the code and bang it into a new, protocol-specific shape. Patrick estimates that, even after four years of unification work, the kernel contains some 10,000 lines of duplicated filtering code. Beyond that, the structures used in the ABI are also used directly in the kernel's internal representation, making that implementation even harder to change. Separating the two would be possible through the addition of a translation layer, but the details involved (including the need to translate in both directions) increase the risk of adding subtle problems. In summary, the iptables ABI has become a serious impediment to further progress in packet filtering. 

Nftables is a chance to dump all of that code and replace it with a much smaller filtering core which should prove to be quite a bit more flexible. With any luck, nftables should last a long time; the virtual machine can be extended in unexpected ways without the need to break the user-space ABI (again). It's smaller size should make it well suited to small router deployments, while its lockless design should appeal to administrators of high-end systems. All told, chances are good that the larger community will eventually see this change as being worthwhile. But not for a while: there are some unfinished pieces in nftables, and the larger discussion has not yet begun. 

(For more information, see [this weblog posting](<http://people.netfilter.org/kaber/weblog/2008/08/20/>) from August, 2008 and [the slides from Patrick's presentation [ODF]](<http://people.netfilter.org/kaber/nfws2008/nftables.odp>) at the Netfilter Workshop).
