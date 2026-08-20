---
title: BPF and security
url: https://lwn.net/Articles/946389/
date: "October 4, 2023"
category: "BPF-Security; Security"
author: "By Jake Edge October 4, 2023 LSSEU"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
October 4, 2023

* * *

[LSSEU](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Security_Summit_Europe>)

The [eBPF in-kernel virtual machine](<https://lwn.net/Articles/740157/>) is approaching its tenth anniversary as part of Linux; it has grown into a tool with many types of uses in the ecosystem. Alexei Starovoitov, who was the creator of eBPF and did much of the development of it, especially in the early going, gave the opening talk at [Linux Security Summit Europe](<https://events.linuxfoundation.org/linux-security-summit-europe/>) 2023 on the relationship between BPF and security. In it, he related some interesting history, from a somewhat different perspective than what is often described, he said. Among other things, it shows how BPF has been both a security problem and a security solution along the way. 

#### Universal assembly

BPF is something like the vi editor, Starovoitov began, people either love it or they hate it, but both are simply sequences of commands. There are lots of definitions of BPF, but the one he wanted to use for the talk is that it is "a universal assembly language". It is the first strictly typed assembly language; there is no "pointer to memory" in BPF, all pointers are to specific types. 

Because it is universal, it goes way beyond the use case of user space telling the kernel what to do; there are now hardware devices that send BPF to the kernel to describe how to use them. There are also user-space-to-user-space applications where the kernel is not involved at all; one application is telling another, perhaps on the other side of the world, what to do. 

[ ![\[Alexei Starovoitov\]](https://static.lwn.net/images/2023/lsseu-starovoitov-sm.png) ](<https://lwn.net/Articles/946563/>)

He is often asked about the difference between BPF and [WebAssembly](<https://webassembly.org/>). BPF is not a sandboxed environment like WebAssembly or JavaScript in a browser; sandboxed environments do not know what code they are going to run so they have to restrict the execution environment. They create a boundary, but that slows down the performance due to all of the run-time checks. 

BPF, instead, is statically verified so there are "practically no run-time checks"; the only checks are for things that the verifier cannot statically determine. The major difference is that the intent of a BPF program is known before it is executed, which is not at all the same for sandboxes, which have to run arbitrary code. He often sees comments (on LWN in particular) about adding WebAssembly to the kernel; his response is that those developers should "bring it in". He believes there is enough room in the kernel for WebAssembly or some other sandbox environment in addition to BPF. 

When eBPF was first proposed in 2013, it was called "internal BPF" (iBPF); it eventually moved to "extended BPF", thus eBPF. In addition, eBPF has itself been extended multiple times; in LLVM, these newer instruction sets are different CPU models that are selected with the `‑mcpu` option using values `v1` to `v4`. The `v4` support was added in July 2023; it is the instruction set that GCC now also supports. 

Starovoitov said that his next slide (number 9 in his [slides](<https://static.sched.com/hosted_files/linuxsecuritysummiteurope20/5a/bpf_and_security.pdf>)) was the most important one in his presentation. Each project should have a mission statement, he said. BPF has two parts to its mission, first, "innovate" and, second, "enable others to innovate". The project continues to satisfy his "thirst for innovation", but it also enables others to do new things. One of his favorite parts of working on BPF is helping people who post on the mailing list with something new they are trying to do; that is "the best moment of being a kernel maintainer". 

He thinks that attitude for the project shows in the growth of the BPF development community within the kernel. He displayed a graph of unique developers by month since the beginning of 2019, which shows a general rise from around 50 to well over 100 in that span, while the BPF team at Meta, which he is part of, has been fairly flat at around ten or fifteen developers over that same span. 

#### Tracing and networking

"BPF has roots in tracing", he said. The first hook that an eBPF program could be attached to was for kprobes and uprobes; tracepoints came next, then function entry (fentry) and exit (fexit). People often think that BPF can do anything within the kernel, but it is actually quite restricted. For tracepoints, the BPF program can read any kernel data, but it cannot modify it at all. Networking BPF programs can read and modify the packet data and drop packets, but they cannot modify the kernel state. The restrictions on what various types of BPF programs can do are based on the use cases for those programs. 

He gave some examples of BPF tracing. Android uses BPF programs to track network usage based on the contacted host, he said. So if a user wants to see their Facebook use on their phone versus YouTube use, say, they are getting that information by way of a BPF program. The [PyPerf](<https://github.com/iovisor/bcc/tree/master/examples/cpp/pyperf>) program uses BPF to profile Python programs. Beyond that, BPF programs can be attached to user-space programs using uprobes such that any invocation of a program will have the BPF program attached. That can be used to see how much time GCC spends processing include files versus compiling the code; each invocation of GCC in a parallel make will be instrumented correctly to gather that data. 

There are multiple networking use cases for BPF as well. He noted that the [express data path](<https://lwn.net/Articles/750845/>) (XDP) feature of the kernel came about as a way to fend off distributed denial of service (DDoS) attacks. At one point, Facebook was under a DDoS attack of 500Gbps that was mitigated by putting a BPF program at the network-driver level using XDP. Absorbing the attack that way provided a 10x improvement over earlier DDoS-mitigation techniques, he said. 

#### Security

The ability to [attach BPF programs to Linux security module (LSM) hooks](<https://docs.kernel.org/bpf/prog_lsm.html>) (also known as BPF-LSM) is a recent addition to the kernel that is being used to "prevent interesting security attacks", Starovoitov said. As with other BPF program types, the BPF programs that can be attached to LSM hooks (or to system calls) have specific capabilities that are different from those of tracing, networking, or other program types. Those programs can read arbitrary kernel data and deny operations, but they can also sleep, which is something new for BPF programs. That means the program can cause a minor fault if the user-space address it is accessing has been swapped out, so it is not possible to evade these hooks by referring to swapped-out memory. 

Unfortunately, the BPF-LSM programs are generally not publicly available, unlike those for, say, tracing and networking. At least 90% of the programs that he knows about in those realms are freely available; in particular, many of the large internet companies are working together on things like DDoS protection, sharing their code, and learning from each other. The same is true in tracing, but the story for BPF-LSM code "is not that good" 

The BPF feature set for networking, tracing, and, even, security is pretty much set at this point; they are 95% done, though, of course, the last 5% takes lots more time, he said. But there are still new things being added to BPF, including the recently landed [BPF for human interface device](<https://lwn.net/Articles/909109/>) (HID) feature. This allows BPF programs to modify how HID devices, such as keyboards and mice, are seen by the kernel, to correct problems (quirks) or change the behavior in some way. 

Starovoitov said that he is excited about the [extensible scheduler class](<https://lwn.net/Articles/922405/>) that would allow BPF programs to perform scheduling functions in order to test out new scheduling algorithms. There are always niche use cases where a more-specialized scheduler is needed, especially in cloud workloads where the schedulers in the virtual machines end up fighting with the hypervisor scheduler. So far, at least, pluggable scheduling using BPF has [been rejected](<https://lwn.net/Articles/939332/>), though that was not mentioned in the talk. 

#### Unprivileged BPF

The original Berkeley Packet Filter (BPF) instruction set was created 30 years ago; it lives on in Linux as "classic BPF" (cBPF) and is used by [`tcpdump`](<https://man7.org/linux/man-pages/man1/tcpdump.1.html>) and [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>). Uses of cBPF are unprivileged, so eBPF followed suit: two of the 32 BPF program types can be used without privileges, and both only allow reading packet data and dropping packets. One of those, `BPF_PROG_TYPE_SOCKET_FILTER`, has been completely unused, he said; all of the other program types always required root privileges. 

That was fine for the first few years of eBPF's existence in the kernel, he said—until 2017\. That was when Jann Horn of [Project Zero](<https://googleprojectzero.blogspot.com/p/about-project-zero.html>) wrote [some BPF code that demonstrated speculative-execution problems](<https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html>), which eventually became known as Spectre v1. Modern CPUs all do speculative execution, but the side effects of their mispredictions still remain in the caches, so they cannot be hidden. As was seen at the end of 2017 (and in the years since), those side effects can be turned into security vulnerabilities. 

The hardware vendors response to these problems was to recommend that any possible branch mispredictions be stopped by adding "load fence" (`lfence`) instructions throughout the code. Microsoft followed this advice and changed its compilers to emit those instructions all over the place; Windows and other tools were then rebuilt. 

The hardware vendors requested that the Linux kernel do the same, but kernel developers had other ideas, he said. The `lfence` instruction is a big hammer, with major performance implications, so it was decided that the kernel would manage speculation by steering it in safe directions rather than turning it off with `lfence`. It took a lot of work to convince Intel and Arm that the technique was feasible, but it resulted in a [better fix for the problem](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f3804203306e098dae9ca51540fcd5eb700d7f40>). The `array_index_nospec()` macro from that patch is used 240 times in the kernel today, some of them in extremely hot paths, such as looking up indexes in the file-descriptor table. The impact of using `lfence` instructions instead would have been huge. 

BPF was used in Horn's exploit, so changes were needed there as well. BPF cannot use the macro directly, but it makes an equivalent change to avoid Spectre v1. Only a few months later, though, Horn came back with a Spectre v2 exploit that used the BPF interpreter, which caused some additional concern about the security story for BPF. The exploit was not actually loading BPF code into the kernel; the speculative execution was using the interpreter on BPF instructions that lived in user space. 

The solution was to avoid having the interpreter code in the kernel executable. BPF code can either be interpreted or run with the just-in-time (JIT) compiler, so the `BPF_JIT_ALWAYS_ON` option was added to always enable the compiler for BPF and remove the interpreter. While BPF was changed to avoid this problem (which is, of course, really in the CPU hardware), he believes that any interpreter in the kernel could be used in this way; there are at least three other interpreters, so the kernel is still not fully safe, Starovoitov said. 

It is interesting how the perception of the BPF JIT compiler has changed over the years, he said. In 2011, there was a [JIT spraying](<https://en.wikipedia.org/wiki/JIT_spraying>) attack against it, which made some kernel developers wonder whether JIT compilation had any place in the kernel. That problem was addressed at the time, but now the JIT compiler has to be enabled to avoid Spectre v2. The BPF developers have also found that the JIT compiler recovers performance that was lost to [retpolines](<https://support.google.com/faqs/answer/7625886>), which are another Spectre v2 mitigation. 

In 2019, the BPF developers decided to get smarter with the verifier to avoid other speculative-execution problems. Daniel Borkmann worked on [verifier changes](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=979d63d50c0c0f7bc537bf821e056cc9fe5abd38>) to detect and avoid these problems. To do so, the verifier simulates both normal and speculative execution, which is "unique in the industry, no other static-analysis tool can do this kind of speculative analysis". 

Along came Spectre v4, which was mitigated with a handful of lines of code in the verifier to sanitize the stack. But other Spectre variants kept showing up, so it was eventually decided to add a configuration option to disable unprivileged BPF completely. The two program types that could be used without privileges were "extremely niche use cases with a number of users less than the number of fingers on a hand"; it simply was not worth the continued pain to the BPF community to keep trying to support that feature, Starovoitov said. The `BPF_UNPRIV_DEFAULT_OFF` option defaults to "on" so that distributions do not allow unprivileged BPF programs, though administrators can override that choice. 

#### `CAP_BPF`

Over the years, there were persistent calls to split out some BPF permissions from the root privileges (actually `CAP_SYS_ADMIN`) needed to perform nearly all BPF actions. The `CAP_PERFMON` capability was added by the perf subsystem, but it has been adopted by BPF as well; it allows reading kernel memory. The `CAP_BPF` capability was added to govern various types of BPF operations; it can be combined with `CAP_PERFMON` to allow loading useful tracing programs or with `CAP_NET_ADMIN` to allow loading useful networking programs—either without requiring `CAP_SYS_ADMIN`. 

There is a perception problem with `CAP_BPF`, however, he said; it is not clear what it is actually meant to govern. Part of the problem is that BPF is not constrained by namespaces; if you can look at kernel memory, you can look at all of kernel memory, not just that in a single container. `CAP_BPF` is meant to work like `CAP_SYS_MODULE`, which is the capability required to load a kernel module; that capability effectively gives permission to crash the kernel because malicious (or buggy) modules can do just that. 

But verifier bugs can lead to BPF programs crashing the kernel, which should be expected as a possibility, but is treated as a security hole instead, Starovoitov said. So every verifier bug gets a CVE, which is a real problem. He noted a mid-September [LWN article](<https://lwn.net/Articles/944209/>) on the topic of "bogus CVEs", which is a problem for the BPF project as well. Bugs are fixed, but get CVEs filed for earlier kernel versions where backports have not been made; sometimes those CVEs even reference the self-test code that BPF runs to ensure the bug remains fixed. The existence of a CVE then creates a panic to fix the older kernel. 

Some security startups are using BPF in strange ways. He noted one unnamed startup that complained about what BPF can do and the dangers to systems inherent in the existence BPF; that was all done to sell the startup's product, of course. The strange piece is that the product was using BPF to protect against all of the BPF problems it was railing against. "In the end, they were saying 'BPF is bad, use BPF to protect from BPF'." 

He began running short of time, so he started quickly moving through the rest of his talk. He noted that one of the few BPF-LSM programs that is open source is one that is used by systemd to prevent the mounting of filesystem types based on allow and deny lists. He showed a few other examples of how systemd is using BPF to enforce various security policies. In some cases, it is using the BPF-LSM hooks, but in others it is using other BPF program types (e.g. networking and tracing) to do its job. 

Starovoitov said that he believes all kernel modules should really be written as BPF programs. The advantages of kernel modules is that they can be written as arbitrary C code, with full access to the kernel's symbols, but that means they can also crash the kernel due to a bug. For BPF programs, safety is built into them via the verifier. In addition, BPF programs are more portable than kernel modules. That portability is an underestimated advantage, especially for companies with large fleets of systems, some long tail of which will have a variety of different kernel versions. 

He closed with a quick note that he thinks the version of C that is used by BPF is a better version of C for kernel programming. The safety that has been built into the flavor of C that can be verified is a better choice for kernel programming overall. His slides showed a few buggy constructs that could be avoided, but he was not able to get into any of the details, though some were [mentioned in a talk](<https://lwn.net/Articles/909095/>) from a year ago. One suspects that may not be an opinion that is widely held outside of the BPF community—at least yet. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Bilbao for LSSEU.]
