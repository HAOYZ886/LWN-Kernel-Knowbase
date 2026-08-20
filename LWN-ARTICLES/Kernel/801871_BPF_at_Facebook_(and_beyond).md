---
title: BPF at Facebook (and beyond)
url: https://lwn.net/Articles/801871/
date: "October 10, 2019"
category: BPF
author: "By Jonathan Corbet October 10, 2019 Kernel Recipes"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 10, 2019

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2019-Kernel_Recipes>)

It is no secret that much of the work on the [in-kernel BPF virtual machine](<https://lwn.net/Articles/740157/>) and associated user-space support code is being done at Facebook. But less is known about how Facebook is actually using BPF. At Kernel Recipes 2019, BPF developer Alexei Starovoitov [described](<https://kernel-recipes.org/en/2019/talks/bpf-at-facebook/>) a bit of that work, though even he admitted that he didn't know what most of the BPF programs running there were doing. He also summarized recent developments with BPF and some near-future work. 

#### Kernels at Facebook

[![\[Alexei
Starovoitov\]](https://static.lwn.net/images/conf/2019/kr/AlexeiStarovoitov-sm.jpg)](<https://lwn.net/Articles/801885/>) Facebook, he began, has an upstream-first philosophy, taken to an extreme; the company tries not to carry any out-of-tree patches at all. All work done at Facebook is meant to go upstream as soon as it practically can. The company also runs recent kernels, upgrading whenever possible. The company can move to a new kernel in a matter of days; this process could be faster, he said, except that it still takes some time to reboot thousands of servers. As of just before the talk, most of the Facebook fleet was running 4.16, with a few 4.11 machines hanging around and some at 5.2. 

He pointed out the lack of long-term-support kernels in the above list. Facebook does not plan to stay with any given kernel for a long time, so the company doesn't care about long-term support. Instead, machines are simply upgraded to whichever kernel is available. Within a given version, though, there can be a fair amount of variation across the fleet; the kernel team evidently backports features into older kernels when the need arises. That can create challenges for applications and, especially, BPF-based applications. 

The first rule of kernel development is "don't break user space"; anything that might cause a user-space program to fail becomes part of the kernel ABI. Performance regressions are included in this rule. Performance problems are easy to create, so Facebook needs a team to track them down. Often, it seems, BPF is fingered as the cause of these problems. 

Starovoitov asked the audience to guess how many BPF programs were running on their laptops, then to run this command: 

```
ls -la /proc/*/fd | grep bpf-prog | wc -l
```

The answer on your editor's system is six, all running from systemd. He was surprised by the answer at Facebook: there are about 40 BPF programs running on each server, with another 100 that are demand loaded. There are many teams within the company writing and deploying these programs; the kernel team doesn't even know about all of these BPF programs. These programs are about evenly split between those attached to kprobes, those attached to tracepoints, and network scheduling-class helpers; about 10% fall into other categories. 

He gave a few examples of performance issues that, at least on the surface, were caused by BPF: 

  * Facebook runs a packet-capture daemon that makes use of a network scheduling-class BPF program; it occasionally spits out a packet for inspection. On new kernels, running that daemon regressed overall system performance by about 1%. It turns out that this daemon uses another BPF program, attached to a kprobe, for a different purpose. The function that probe attaches to didn't exist in the newer kernel, causing the daemon to conclude that BPF as a whole was broken; it then fell back to an older, slower method for packet capture. Kprobes are not a stable ABI, Starovoitov said, but when kernel developers change a function kprobe usage can still require somebody to investigate the resulting breakage. 
  * The number-one performance-analysis tool at Facebook is a profiling daemon that attaches BPF programs to tracepoints and kprobes in the scheduler and beyond. On new kernels, it caused a 2% performance regression, manifesting as an increase in software-interrupt time. It turns out that, in the 5.2 kernel, setting a kprobe causes the text section to be remapped from 2MB huge pages to normal 4KB pages, with a resulting increase in TLB misses and decrease in performance. 
  * There is a security monitoring daemon that sets BPF programs on three kprobes and one kretprobe. It runs at low priority, waking up every few seconds and consuming about 0.01% of the CPU. This daemon was causing huge latency spikes for the high-priority database application. Some tracing work showed that, on occasion, a `memcpy()` call in the database could stall for as much as 1/4 second while this daemon was reading its `/proc/_pid_ /environ` file. Much more tracing showed that this daemon was acquiring the `mmap_sem` lock when reading that `/proc` file, then being scheduled out for long periods of time, blocking page faults in the main application. The root cause was a basic priority-inversion issue; raising the security daemon's priority prevents this problem. 

The takeaway from all of these episodes — and especially the last one — is that the best tool for tracking down BPF-related performance regressions is BPF. 

#### Current and future BPF improvements

Another kind of problem results from how BPF programs are built. A user-space application will contain one or more BPF programs to be loaded into the kernel. These programs are written in C and compiled to the BPF virtual machine instruction set; this compilation happens on the target system. To ensure that the compilation can be done consistently, a version of the LLVM compiler is embedded in the application itself. This makes the applications big, and the compilation process can perturb the main workload on the target system. The compilation can also take a long time, since it is done at a low priority; several minutes to compile a 100-line program is not unusual. The system headers needed to understand kernel data structures may be missing from the target system, creating compilation failures. It is a pain point, he said. 

The solution to this problem is to be found in the "compile once, run everywhere" work that reached a milestone with the 5.4 kernel. It uses the BPF type format (BTF) data describing kernel data structures that was created for just this purpose. With BTF provided by the kernel, there is no longer any need to ship kernel headers around; instead, the bpftool utility just extracts the BTF data and creates a "monster header file" on the target system. An LLVM built-in function has been added to preserve the offsets into structures used by BPF programs; those offsets are then "relocated" at load time to match the version of the structure used in the target kernel. 

A number of other interesting projects have made progress in 2019, he said. Support for [bounded loops](<https://lwn.net/Articles/794934/>) in the verifier was added to 5.3 after two years of work. BPF programs can now [manage concurrency with spinlocks](<https://lwn.net/Articles/779120/>), with the verifier proving that these programs will not deadlock. Dead-code elimination has been added, and scalar precision tracking as well. 

Starovoitov said that people often complain that the BPF verifier is painful to deal with. But, he said, it is far smarter than the LLVM compiler, and a number of advantages come from that, starting with the ability to prove that a program is safe to load into the kernel. The verifier is also able to perform far better dead-code elimination than LLVM can. 

In the future, the verifier is set to get better by making more use of the available BTF data. Every program type, for example, must implement its own boilerplate functions to provide (and check) access to the context object passed to the programs themselves. This code bloats the kernel, he said, and tends to be prone to bugs. With BTF, those functions will no longer be necessary; the verifier can use the BTF data to check programs directly. That will enable the removal of 4,000 lines of code, he said. 

He concluded by saying that BPF development is "100% driven by use cases"; the way to shape its future direction is to show the ways in which new features can be useful. Even better, of course, is to hack new extensions and to share them with the community. 

[Your editor thanks the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event.]
