---
title: Taming the BPF superpowers
url: https://lwn.net/Articles/870269/
date: "September 29, 2021"
category: BPF
author: "By Jonathan Corbet September 29, 2021 LPC"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 29, 2021

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2021-Linux_Plumbers_Conference>)

Work toward the signing of BPF programs has been finding its way into recent mainline kernel releases; it is intended to improve security by limiting the BPF programs that can be successfully loaded into the kernel. As John Fastabend described in his ["Watching the super powers" session](<https://linuxplumbersconf.org/event/11/contributions/947/>) at the [2021 Linux Plumbers Conference](<https://linuxplumbersconf.org/>), this new feature has the potential to completely break his tools. But rather than just complain, he decided to investigate solutions; the result is an outline for an auditing mechanism that brings greater flexibility to the problem of controlling which programs can be run. 

The kernel has had the ability to enforce signatures on loadable modules for years, so it makes sense to consider creating the same mechanism for BPF programs. But, while kernel modules and BPF programs look similar — both are code loaded into the kernel from user space, after all — there are some significant differences between them. The safety of kernel modules is entirely dependent on the diligence of developers. They are built and distributed via the usual channels, are tied to specific kernel versions, and can last for years; they present a stable API to user space. BPF programs, instead, benefit from safety built into (and enforced by) the loader. They are often dynamically built and optimized, they are patched at run time to avoid being tied to kernel versions, and they have a different lifetime; often, they are created on the fly and quickly thrown away. These differences suggest that the same signing mechanism might not work equally well for both types of program. 

Fastabend covered the BPF signing scheme; curious readers can find a more complete description in [this article](<https://lwn.net/Articles/853489/>). In short: BPF program loading is a complicated, multi-step process involving numerous system calls; the signature is meant to cover this entire process. [![\[John Fastabend\]](https://static.lwn.net/images/conf/2021/lpc/JohnFastabend-sm.png)](<https://lwn.net/Articles/870272/>) That is done by loading yet another BPF program to handle the process; this mechanism is mostly implemented, but there are some details left to be worked out. 

There are certainly advantages to this mechanism, he said. If Alice and Bob have signed BPF programs, they can use them as usual. If Eve comes along with an unsigned program meant to eavesdrop on the kernel, that program will not be loaded and Eve will be frustrated. But there is also a cost: if Alice is generating programs on the fly, those programs will not be signed and will no longer be loadable. The keys used to sign programs should not be present on the system, so signing cannot be done on the fly and Alice's workflow will be blocked. Alice, too, will be frustrated despite being a legitimate user. 

This is not just a hypothetical case; a lot of tooling works that way now. Perhaps the best-known example is [bpftrace](<https://github.com/iovisor/bpftrace>), but it's not the only one. The [P4](<https://opennetworking.org/p4/>) system defines a domain-specific language for the management of networking data paths. Some of Fastabend's work on [Cilium](<https://cilium.io/>) is aimed at run-time optimization of BPF programs. [PcapRecorder](<https://doc.omnetpp.org/inet/api-current/neddoc/inet.common.packet.recorder.PcapRecorder.html>) is an [XDP](<https://en.wikipedia.org/wiki/Express_Data_Path>)-based clone of the venerable [tcpdump](<https://www.tcpdump.org/>) utility. And so on. None of these tools can work in an environment where BPF programs must be signed. 

A lot of the security goals can be achieved, he said, by just making use of the [fs-verity](<https://lwn.net/Articles/790185/>) mechanism supported by Linux now. With fs-verity, read-only files can be signed and the kernel will check the signature on each access. If the file has been corrupted somehow, the signature will not match and access to the file will be blocked. So one thing that can be done is to use fs-verity to sign the program that loads BPF programs into the kernel. The system will automatically ensure that this program is not tampered with, and the set of keys that can create valid signatures can be restricted. 

But it is possible to go further than that, Fastabend said. Using some sort of policy engine, which could be another BPF program or a Linux security module, the kernel can look at the key that was used to sign any given program and associate a set of privileges with it. At its most basic, there could be a single "can load BPF programs" privilege, which would be similar in effect to attaching the `CAP_BPF` capability to the program. The system could be more fine-grained than that, even, by controlling actions like access to maps. With this sort of mechanism, he said, signature checking on the BPF objects themselves will be unnecessary. 

Consider the case where Alice's BPF-using process is somehow corrupted at run time. Signing of BPF programs will not save the system in this case; the corrupted user-space code can still do things like change values in BPF maps, change the attachment points for programs, and more. In other words, signing a BPF program gives little assurance that said program will run correctly in a hostile environment. A more flexible policy mechanism might do better, though, and could block attempts by a program to exceed its boundaries. Perhaps unsigned programs could be allowed to load, but they would not have the ability to write to user space or access kernel memory, for example. Access to pinned maps could be denied as well. 

This mechanism is not yet implemented, but he has some ideas about how it could be done. The LLVM compiler can attach attributes to objects; it could be taught to record all of the helper functions that a program calls, all of its map operations, and so on. The BPF verifier would then confirm that the program stays within those limits, and the supervisor mechanism could allow or deny a specific program based on the attributes. All that's left is to figure out how all this would actually work. 

Fastabend concluded by reiterating his goal of ensuring that dynamically generated BPF programs keep working. Program signing seems like the wrong solution; it is only useful in cases where the signed programs won't change. With an appropriate set of policy rules, it should be possible to safely allow a system to run unsigned BPF programs. In the brief discussion period that followed, Alexei Starovoitov (the author of the existing signing work) noted with enthusiasm that there are many other types of permissions that could be added. The maximum number of instructions allowed in a program would be one example. So there appears to be interest in this idea, but the real proof, as always, is in the code. 

The [video of this session](<https://www.youtube.com/watch?v=xj0PBFjLm1U&t=5730s>) is available on YouTube.
