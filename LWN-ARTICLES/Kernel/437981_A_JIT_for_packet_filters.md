---
title: A JIT for packet filters
url: https://lwn.net/Articles/437981/
date: "April 12, 2011"
category: "BPF; Networking-Packet filtering; Nftables"
author: "By Jonathan Corbet April 12, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 12, 2011

The [Berkeley Packet Filter](<https://secure.wikimedia.org/wikipedia/en/wiki/Berkeley_Packet_Filter>) (BPF) is a mechanism for the fast filtering of network packets on their way to an application. It has its roots in BSD in the very early 1990's, a history that was not enough to prevent the SCO Group from claiming ownership of it. Happily, that claim proved to be as valid as the rest of SCO's assertions, so BPF remains a part of the Linux networking stack. A recent patch from Eric Dumazet may make BPF faster, at least on 64-bit x86 systems. 

The purpose behind BPF is to let an application specify a filtering function to select only the network packets that it wants to see. An early BPF user was the tcpdump, which used BPF to implement the filtering behind its complex command-line syntax. Other packet capture programs also make use of it. On Linux, there is another interesting application of BPF: the "socket filter" mechanism allows an application to filter incoming packets on any type of socket with BPF. In this mode, it can function as a sort of per-application firewall, eliminating packets before the application ever sees them. 

The original BPF distribution came in the form of a user-space library, but the BPF interface quickly found its way into the kernel. When network traffic is high, there is a lot of value in filtering unwanted packets before they are copied into user space. Obviously, it is also important that BPF filters run quickly; every bit of per-packet overhead is going to hurt in a high-traffic situation. BPF was designed to allow a wide variety of filters while keeping speed in mind, but that does not mean that it cannot be made faster. 

BPF defines a virtual machine which is almost Turing-machine-like in its simplicity. There are two registers: an accumulator and an index register. The machine also has a small scratch memory area, an implicit array containing the packet in question, and a small set of arithmetic, logical, and jump instructions. The accumulator is used for arithmetic operations, while the index register provides offsets into the packet or into the scratch memory areas. A very simple BPF program (taken from the [1993 USENIX paper [PDF]](<http://www.tcpdump.org/papers/bpf-usenix93.pdf>)) might be: 

```
ldh	[12]
    	jeq	#ETHERTYPE_IP, l1, l2
        l1:	ret	#TRUE
        l2:	ret	#0
```

The first instruction loads a 16-bit quantity from offset 12 in the packet to the accumulator; that value is the Ethernet protocol type field. It then compares the value to see if the packet is an IP packet or not; IP packets are accepted, while anything else is rejected. Naturally, filter programs get more complicated quickly. Header length can vary, so the program will have to calculate the offsets of (for example) TCP header values; that is where the index register comes into play. Scratch memory (which is the only place a BPF program can store to) is used when intermediate results must be kept. 

The Linux BPF implementation can be found in `net/core/filter.c`; it provides "standard" BPF along with a number of Linux-specific ancillary instructions which can test whether a packet is marked, which CPU the filter is running on, which interface the packet arrived on, and more. It is, at its core, a long `switch` statement designed to run the BPF instructions quickly. This code has seen a number of enhancements and speed improvements over the years, but there has not been any fundamental change for a long time. 

Eric Dumazet's [patch](<https://lwn.net/Articles/437884/>) is a fundamental change: it puts a just-in-time compiler into the kernel to translate BPF code directly into the host system's assembly code. The simplicity of the BPF machine makes the JIT translation relatively simple; every BPF instruction maps to a straightforward x86 instruction sequence. There are a few assembly language helpers which help to implement the virtual machine's semantics; the accumulator and index are just stored in the processor's registers. The resulting program is placed in a bit of `vmalloc()` space and run directly when a packet is to be tested. A [simple benchmark](<https://lwn.net/Articles/437986/>) shows a 50ns savings for each invocation of a simple filter - that may seem small, but, when multiplied by the number of packets going through a system, that difference can add up quickly. 

The current implementation is limited to the x86-64 architecture; indeed, that architecture is wired deeply into the code, which is littered with hard-coded x86 instruction opcodes. Should anybody want to add a second architecture, they will be faced with the choice of simply replicating the whole thing (it is not huge) or trying to add a generalized opcode generator to the existing JIT code. 

An obvious question is: can this same approach be applied to iptables, which is more heavily used than BPF? The answer may be "yes," but it might also make more sense to bring back the [nftables](<https://lwn.net/Articles/324989/>) idea, which is built on a BPF-like virtual machine of its own. Given that there has been some talk of using nftables in other contexts (internal packet classification for packet scheduling, for example), the value of a JIT-translated nftables could be even higher. Nftables is a job for another day, though; meanwhile, we have a proof of the concept for BPF that appears to get the job done nicely.
