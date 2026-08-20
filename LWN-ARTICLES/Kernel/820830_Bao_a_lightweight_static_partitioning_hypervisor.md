---
title: "Bao: a lightweight static partitioning hypervisor"
url: https://lwn.net/Articles/820830/
date: "May 20, 2020"
category: Virtualization
author: "By Jonathan Corbet May 20, 2020 OSPM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 20, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

Developers of safety-critical systems tend to avoid Linux kernels for a number of fairly obvious reasons; Linux simply was not developed with that sort of use case in mind. There are increasingly compelling reasons to use Linux in such systems, though, leading to a search for the best way to do so safely. At the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), José Martins described [Bao](<https://github.com/bao-project/bao-hypervisor>), a minimal hypervisor aimed at safety-critical deployments. 

The actual target, he began, is "mixed-criticality" systems in which multiple software stacks run in parallel with each other; some of those stacks are safety-critical, while others are not. For example, a system could have a user interface running on Linux alongside the safety-critical application that it controls. There is an industry trend toward consolidating systems in this way driven by power considerations and the availability of processors with numerous CPUs. 

Virtualization is naturally interesting for the developers of such systems; it minimizes the effort required to port systems and eases the integration within them. Good virtualization provides fault isolation, preventing failures in one [![\[José Martins\]](https://static.lwn.net/images/conf/2020/ospm/JoseMartins-sm.png)](<https://lwn.net/Articles/820832/>) part of the system from interfering with others. Developers want the usual things from such a system: good performance, realtime guarantees, and strong security. 

Martins spent some time looking at solutions like Xen and KVM. They were not designed for this kind of use case, but they end up being used anyway. Neither is an optimal solution; they use virtualized I/O mechanisms that add a lot of overhead, and their code bases are large and hard to audit. 

Instead, he said, there is a role here for a static partitioning hypervisor, which can be seen as a thin configuration layer that divides up a system's resources. Under a system like this, there is a one-to-one mapping between virtual CPUs and physical CPUs, so there is no contention for CPU time. Devices are mapped directly into the guests, avoiding any added I/O overhead. Perhaps the best-known hypervisor of this type is [Jailhouse](<https://github.com/siemens/jailhouse>), but that didn't meet Martins's needs; it depends on a Linux "root cell" to run the whole show, its boot time is relatively long, and there is still a big code base to audit. The [Xen Dom0-less](<https://xenproject.org/2019/12/16/true-static-partitioning-with-xen-dom0-less/>) project can do direct device assignment, which is nice, but it falls short in other ways. 

So Martins set out to create Bao as a "type-1 bare-metal hypervisor" with a one-to-one CPU mapping. It doesn't depend on any sort of privileged virtual machine or operating system to boot. Bao provides a simple inter-VM communication mechanism based on shared memory and virtual interrupts. It depends on hardware assistance for many of its functions, including second-stage address translation, an I/O memory-management unit, and virtual interrupts. Bao can use huge pages to reduce translation lookaside buffer pressure and page-table memory use; it is also able to perform [cache coloring](<https://en.wikipedia.org/wiki/Cache_coloring>) for memory allocations to avoid low-level cache interference between machines. 

Bao currently targets the Armv8 architecture. There is a RISC-V port, but the virtualization specification for RISC-V is not ready, so this port is not interesting yet. It can run a number of guests, including bare-metal applications, Linux, Android, and various realtime operating systems. 

Ideally, he said, Bao would just be a configuration layer that does its work and gets out of the way, but the hardware does not support this mode of operation. Interrupts, for example, have to be mediated through the hypervisor, which is unfortunate since that increases latency. The I/O memory-management unit has a limited number of stream registers, and doesn't cover all devices on some platforms. There is no partitioning mechanism for memory cache on Arm, so the hypervisor must handle isolation via cache coloring. 

The system is implemented in about 7,000 lines of code, and requires 50KB of memory on the target system. That is "somewhat small", he said, but he is working to get it smaller. Run-time memory requirements add up to about 250KB. Benchmark runs show that the hypervisor adds an execution-time overhead of about 2%. Turning on cache coloring increases that overhead, since that feature is incompatible with the use of huge pages. Interference tests currently show a significant amount of degradation caused by activity in other virtual machines; cache coloring helps but does not completely solve the problem. 

Another issue is interrupt latency, which increases significantly due to the need for a round-trip through the hypervisor. There is a fair amount of cross-VM interference caused by interrupts as well; again, cache coloring helps, especially if it is used within the hypervisor too. He has found a way to map interrupts directly into guests, something that is made possible by the one-to-one CPU mapping. That increases the overhead for interrupts intended for the hypervisor itself, but those are relatively rare. 

Current work includes adding support for trusted execution environments. The other approaches out there share the trusted code across virtual machines, which is not ideal. Arm is adding support for a "trusted hypervisor" mode but, for now, complex workarounds are required. Martins said that the "dual-world" approach used in this area is not inherently secure; a lot of code has to be added to the secure side, bringing the same old problems with it. It is better, he said, to limit the secure world to core security primitives; he is trying to do that by avoiding [TrustZone](<https://developer.arm.com/ip-products/security-ip/trustzone>) completely and dedicating a virtual CPU to trusted work. This involves allowing multiple virtual CPUs to run on a single physical CPU. 

Overall, he concluded, Bao has turned out to be a good fit for the intended use case.
