---
title: Virtual machine scheduling with BPF
url: https://lwn.net/Articles/974363/
date: "May 22, 2024"
category: BPF
author: "By Daroc Alden May 22, 2024 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Daroc Alden**  
May 22, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Vineeth Pillai gave a remote talk at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) explaining how BPF could be used to improve the performance of virtual machines (VMs). Pillai has [ a patch set](<https://lwn.net/ml/linux-kernel/20240403140116.3002809-1-vineeth@bitbyteword.org/>) designed to let guest and host machines share scheduling information in order to eliminate some of the overhead of running in a VM. The assembled developers had several comments on the design, but seemed overall to approve of the prospect. 

VMs have a variety of potential performance footguns, but a particularly persistent one is "double scheduling". When using KVM, the implementation of a virtual machine hypervisor in the Linux kernel, virtual CPUs correspond to threads. This means that the host system's scheduler will assign the thread for a given virtual CPU to a physical CPU, and then the guest system's scheduler will assign threads to those virtual CPUs. This results in a certain amount of unavoidable overhead just from running two schedulers, but it also increases the amount of jumping around between physical cores that processes on the guest need to tolerate. 

This problem can be partially mitigated using CPU pinning, but that is a manual solution that still doesn't address the more subtle aspect of double scheduling: that useful information is lost between the two schedulers. Pillai and his collaborator Joel Fernandes have been working on a solution that allows the guest and host to share scheduling information, allowing the host scheduler to make more intelligent decisions about where to put vCPU threads and how to schedule them. 

To make this work, their proposed system would use memory shared between the guest and the host. The guest runs a `pvsched` driver that allocates the necessary memory and shares it with the host. The driver then streams relevant scheduling information into that memory, and reads any information that the host wants to provide in return. The most recent version of the patch set is version 2, published in April, but Pillai is already working on a version 3 to address comments from the KVM maintainers. 

On the host side, this scheme is integrated into the scheduler using BPF. The BPF program reads information from KVM, including the PIDs and assigned physical CPUs of the virtual CPU threads, and the location of the guest's shared memory, from a BPF map. The BPF program can then make scheduling decisions, and call hooks in the scheduler to override its decisions about how to schedule the virtual CPUs, Pillai said. 

David Vernet asked whether it would make sense to define a (supposedly immutable) user-space API around the `pvsched` driver, or whether it would make sense to do communication wholly over a BPF channel. BPF interfaces are not considered part of the kernel's API stability promises — but the KVM interface to guest VMs is. Pillai responded that the idea of using a BPF-to-BPF channel makes sense. Vernet later suggested adding a new BPF map type that goes directly between the host and the guest. Pillai concurred with the idea of a guest-to-host map type. 

Pillai said they did have one question about the design for the assembled developers — should their patch set use `struct_ops` callbacks or raw tracepoints to hook into the KVM subsystem? Vernet questioned whether Pillai was proposing calling kfuncs (to manipulate the scheduler) from inside a tracepoint. Pillai agreed that he was. Steven Rostedt pointed out that calling kfuncs from some tracepoints could deadlock the scheduler, so you would need some kind of allowlist of which tracepoints could be used this way. 

Vernet agreed, suggesting that you could use a per-CPU variable to check whether the BPF function associated with a tracepoint was being called from one of the allowed locations. Rostedt responded by asking whether this was something that the verifier could check. Vernet indicated that this was not yet possible — and that it was an example of the need for more granularity around deciding how kfuncs can be called, as he suggested in [his earlier session on polymorphic kfuncs](<https://lwn.net/Articles/974102>). 

Rostedt pointed out that an advantage of using tracepoints is that there would be no need to add anything to the KVM subsystem to support it, since `vmm_enter()` and `vmm_exit()` (functions that bracket any code being run in the virtual machine) already have tracepoints. Pillai clarified that those tracepoints are too late for their purposes. Rostedt suggested that it could make sense to ask the KVM maintainers whether those could be moved. 

The audience had some concerns about the entire idea of opening up the ability to override the scheduler in this way. Rostedt noted that once the possibility exists for BPF to change the scheduler properties of a thread, there will be more uses for that than just this KVM change. Vernet said that this was a question for the scheduling folks, pointing out that user space can already do it "but they probably don't want BPF setting scheduler knobs". 

Despite some questions about the implementation, everyone seemed receptive to the idea of eliminating the double-scheduling problem. When Pillai finishes the third version of the patch set, we will see whether the KVM and scheduler maintainers feel the same way.
