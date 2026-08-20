---
title: "What's brewing in CXL"
url: https://lwn.net/Articles/1072858/
date: "May 19, 2026"
category: Compute Express Link CXL
author: "By Jonathan Corbet May 19, 2026 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 19, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

[Compute Express Link (CXL)](<https://en.wikipedia.org/wiki/Compute_Express_Link>) is a technology intended to enable the provision of "memory nodes" in data centers that provide (possibly shared) memory to nearby CPUs. It has, Dan Williams said at the beginning of his memory-management-track session on the topic at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), ""been making memory-management problems worse since 2021"". He used the session to provide an overview of the ways in which CXL can be expected to extend that record into the future. 

CXL, he began, is a way of providing memory over the PCIe bus; it generally offers latency that is worse than accessing memory on remote NUMA nodes. The strength and challenge of CXL is that it is highly configurable, allowing, for example, the setup of interleaved memory access that improves performance. One problem is that, while the kernel can control some access to CXL memory, the system firmware wants to play with things as well. Another challenge comes with the hot-plug nature of CXL memory; if it goes away, a portion of the system's RAM can disappear. 

[![\[Dan Williams\]](https://static.lwn.net/images/conf/2026/lsfmm/DanWilliams-sm.png)](<https://lwn.net/Articles/1072861/>) The CXL standard is evolving quickly, and manufacturers are putting out hardware with interesting deviations from the adopted standards. The kernel is following something like the ACPI "code-first" policy with regard to these changes and [documenting them](<https://docs.kernel.org/driver-api/cxl/conventions.html>) in the kernel tree. The hope is to send a message to manufacturers: ""somebody broke it this way, please break yours the same way"". 

Error handling is an evolving area as well. CXL protocol errors are reported to the kernel as PCIe internal errors, but they are handled through a side channel so that the PCIe core code does not need to deal with them. The CXL code is introducing kernel panics in its error-handling paths; that is not something that is normally appreciated in kernel code but, Williams said, the firmware is going to panic the system anyway in such situations. 

Accelerator support (memory-to-memory compression, for example) is close to landing in the kernel. It turns out that accelerators are relatively simple to support. Another development area is [vfio-cxl](<https://lwn.net/ml/all/20251209165019.2643142-1-mhonap@nvidia.com/>), which is a mechanism to allow exporting CXL accelerators to virtual machines. 

Dynamic capacity has long been a dream for system designers, Williams said. One buys a lot of DIMMs, puts them into a box, runs a cable, and a lot of hosts can then map that memory. But then the kernel has to somehow make that memory available to user space. The plan is to use [device DAX](<https://docs.kernel.org/arch/powerpc/vmemmap_dedup.html>) as the interface, but there are questions about how to create private nodes for dedicated memory (a topic that would return the following day). There needs to be integration with [guest_memfd](<https://lwn.net/Articles/949277/>) as well. 

What is _not_ brewing in CXL support? One non-development area is error isolation; if a CXL host bridge is lost, all associated devices will fail, and terabytes of system RAM may vanish. It is hard to envision a way in which the system can survive such an event, at least when memory is involved. Error isolation for accelerator users might be more feasible. There is also no work currently on supporting peer-to-peer operations on CXL devices, but ""somebody will want it someday"". Finally, CXL encryption, which he described as ""another bucket of acronyms"", is also not expected to be supported in the near future. 

The session concluded there, with no real discussion among those present.
