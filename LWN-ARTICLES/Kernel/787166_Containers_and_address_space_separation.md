---
title: Containers and address space separation
url: https://lwn.net/Articles/787166/
date: "May 1, 2019"
category: "Memory management-Address-space isolation"
author: "By Jake Edge May 1, 2019 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 1, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

James Bottomley began his talk at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM) by noting that the main opposition to his ideas was not present at the summit, which was likely to mean the ideas got a much easier reception than they would have otherwise. In particular, Peter Zijlstra and Ingo Molnar expressed some strong reservations to the work that Bottomley's colleague Mike Rapoport [posted](<https://lwn.net/ml/linux-kernel/1556228754-12996-1-git-send-email-rppt@linux.ibm.com/>) recently; none of those three were in attendance at LSFMM. The idea is to use address spaces to reduce the attack surface available to virtual machines (VMs) and containers such that kernel bugs of various sorts have less reach on multi-tenant systems. 

Bottomley has been working with Rapoport on the idea for the container use case, but there are others, from Google and Oracle, who are trying to solve the same problems for VMs. Address spaces are the oldest and most secure mechanism for keeping tenants separate from one another, he said. Separating processes into their own address spaces is what was used to support multi-user systems, so there is around 50 years of history there. Part of the reason to extend the idea for VMs and containers is that address spaces have proven to work well as a security measure. 

[ ![\[James Bottomley\]](https://static.lwn.net/images/2019/lsf-bottomley-sm.jpg) ](<https://lwn.net/Articles/787209/>)

For a KVM system, the security of the guest OS depends on the fact that the hypercalls make up a pretty tiny footprint within the host kernel. That interface is all that the guest OS has access to in the host. Prior to Spectre and Meltdown, that boundary would be enough for security but now programs can observe what others are doing using the flaws. So KVM developers would like to find ways to ensure that the guest cannot access anything else in the host kernel (even speculatively) by unmapping its code and by not providing access to the address space of its data. 

Moving on to containers, he said that one of the big problems in container security is the kernel footprint that containers use. In the simplest case, containers have access to the entire system-call interface of the kernel, which means they are exposed to a "broad swath of the kernel". If you listen to VM people, he said, they will claim that the access to all of the 360 or so system calls is what makes containers inherently far less secure than VMs. But if you [measure what is actually being used](<https://lwn.net/Articles/773976/>) by containers, "that is bollocks"; generic containers are something like two to five times less secure than hypervisors (in terms of the amount of kernel code traversed), not the hundreds of times that VM proponents like to suggest. 

There is a "semantic gap" between the system calls that a guest VM can make and the code that actually gets executed in the host kernel. The security of VMs is partly based on the fact that it is actually difficult to create an exploit at the system-call level that will translate across the hypercalls into the host kernel. So there is lots of exposed code in the guest OS, but a great deal less in the host OS because of the hypervisor interface. Container developers would like to get back the properties that the semantic gap provides, without reverting to having the gap itself, since it is the lack of this gap that gives containers much of their power. 

But in order to reduce the kernel footprint, projects like [gVisor](<https://github.com/google/gvisor>) and [Nabla Containers](<https://nabla-containers.github.io/>) create sandboxes to handle some of the kernel work in user space. For gVisor, the kernel calls have been rewritten in Go, while Nabla Containers took the approach of reducing the projection of the containers into the kernel by restricting which system calls can be made and emulating the rest in user space. 

The price for those approaches is a "massive semantic gap", he said. It has high security properties, but it causes problems for memory management and elsewhere. He is looking to thin out the sandbox; ideally via separate address spaces for each container or VM. If every kernel namespace could have objects that were private to that namespace, that would go a long way toward solving this problem, he said. It is primarily a protection for multi-tenant systems, so he is not suggesting that the feature would be turned on everywhere. 

Trond Myklebust asked how these ideas differed from a microkernel architecture. In a microkernel, the filesystem driver and the block device driver would each be in their own address spaces, Bottomley said, so sharing filesystems between containers would be problematic because the two containers would be accessing the same filesystem address space. What he is looking for is to separate address spaces on a per-tenant basis, not per subsystem. 

Another attendee noted that in a microkernel, a fault in the filesystem driver, for example, only affects that component, not the rest of the system. The architecture will make it difficult to separate a shared filesystem and all of its data structures; it is a question of which resources you are protecting. 

Bottomley believes that two unrelated containers sharing a filesystem is a niche use case in the cloud. Since most don't do it, we can afford to make them pay a huge penalty when they do, he said. In terms of fault isolation, a microkernel will simply restart a failing driver, but what is needed in the multi-tenant case is to kill the tenant that caused the fault. The idea is to punish a malicious tenant. The difference is that a microkernel provides an address space per subsystem, while he is looking for an address space per tenant; the design goals and security properties are different in the two cases. 

The question of performance was raised; can the performance that will be lost by doing this address-space separation ever be recovered? Bottomley believes that it is a hardware problem. He noted that virtualization was slow initially, so the hardware vendors stepped up to make it faster; the same could happen here. Beyond that, this feature is for security; anything is acceptable in the name of security, he (half) joked. 

Matthew Wilcox said that he did not buy the semantic gap argument. He noted that in a previous life as a Java programmer he had a colleague who was able to accidentally corrupt the BIOS in his machine with a Java program. Bottomley agreed; the semantic gap argument is a form of security through obscurity, he said. There are two main approaches to providing security for containers: either guarding against malicious system calls, which sometimes works and sometimes doesn't, or emulating system calls in user space. 

Returning to the shared-filesystem question, Ted Ts'o noted that often the base images for containers (such as RHEL) are shared. Bottomley said that typically those are shared in a read-only fashion, which is a solved problem. Read-only pages in the page cache can be shared, but sharing read-write pages is not a common cloud-container pattern.
