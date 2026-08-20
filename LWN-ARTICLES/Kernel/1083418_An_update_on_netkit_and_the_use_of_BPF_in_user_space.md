---
title: An update on netkit and the use of BPF in user space
url: https://lwn.net/Articles/1083418/
date: "July 24, 2026"
category: "BPF-Networking"
author: "By Daroc Alden July 24, 2026 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Daroc Alden**  
July 24, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Daniel Borkmann led a session at the 2026 [ Linux Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) about the progress that has been made with netkit, the subsystem that allows virtual machines (VMs) running on Linux to perform networking efficiently. When that did not fill the full time, he went on to discuss his idea for using BPF to live-patch user-space applications. While netkit is making progress, and can now support zero-copy receipt of packets into a VM in a network namespace, the idea of using BPF for patching user-space programs remains entirely speculative. 

There have long been ways for VMs to accelerate their networking or access physical networking devices. Those approaches, however, have not been compatible with the use of network namespaces. That combination is needed by [ KubeVirt](<https://kubevirt.io/>), a Kubernetes setup that uses virtual machines for isolation and network namespaces to set networking policies. Eventually, Borkmann would like to let virtual machines do true zero-copy networking. A critical step toward that is the recently [ merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d04686d9bc86432ea3008d5f358373d8466d1943>) support for queue leasing. 

Queue leasing enables zero-copy networking by allowing the virtual machine manager (VMM) to create a queue on the VM's virtual networking device, and then bind it to a real queue on a host device. Right now, that is only working for receive queues — meaning that when a packet arrives to the physical network interface card (NIC), the driver can place it directly in the memory that corresponds to the virtual NIC's queue, and therefore have it be accessible to the VM directly. 

The support for receive-queue leasing is fairly robust; it already works with other performance optimizations such as huge pages or [ BIG TCP](<https://lwn.net/Articles/884104/>), provided that the physical NIC supports them. There is a need for some self-tests to ensure that these things keep working, Borkmann said. On the transmit side, there are some additional complications that will be more work to address. 

The underlying design for transmit-queue leasing is nearly identical: allowing the VM to place outgoing packets directly in memory accessible to the physical NIC. John Fastabend asked whether the transmit side would need a BPF program to enforce policy. Borkmann replied that, for netkit, there would be such a BPF program installed even in the absence of queue leasing. The thing is, not all physical devices support the kind of direct memory access that would be needed for transmit-queue leasing. There was a patch from Bobby Eshleman ([merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7a348a95f696d20f15c776de4df8b4415bcf3d77>) on May 18), Borkmann said, that would extend the device memory transmit configuration property to indicate whether the physical device supports this kind of access; then netkit just needs to be extended to pass through that property from the underlying physical device. 

Allowing VMs to pass data directly to devices like this would not allow them to circumvent network namespace policies because the packet header still goes through the normal process, including any BPF programs that have been set up to filter or redirect packets. It is only the data part of the packet — the part that is expensive to copy — that is provided directly. Once transmit-queue leasing can be made to work, it should be suitable to most use cases. 

One audience member asked about which component was supposed to be able to set up receive-queue leasing, and what was preventing a VM from abusing the ability to exhaust the host device's number of supported queues. For his use case, Borkmann said, it is [ Cillium](<https://cilium.io/>) that sets up the queue leases. The correspondence is always set up by some software running on the VMM; the VM can't simply decide that it gets to have a queue. The same person also asked why it was necessary to have a separate transmit queue per VM; for the receive-queue side, it's needed because incoming packets may need to be steered to different VMs, but he did not think that it was necessary for transmit queues. 

It might be possible to use a shared transmit queue, Borkmann said, but he had previously seen bugs where, when express data path (XDP) assigned a queue mapping, it caused problems with the state of queues on the physical NIC. Keeping things separate prevents interference of that kind from occurring. The audience member then had some additional suggestions for how the transmit side could be improved which Borkmann promised to look into. 

#### BPF for user-space live patching

The other topic that Borkmann wanted to discuss with the assembled kernel contributors was the possibility of using BPF to live-patch user-space programs. He got the idea from Fastabend's [ earlier talk](<https://lwn.net/Articles/1081546/>), noting that sometimes there are user-space applications that are just as hard to patch against security vulnerabilities in a timely manner as the kernel. What if the kernel could generate trampolines or just-in-time compiled code that runs directly in user-space? That could allow the same kind of function-argument validation or return-value alteration. Such a facility could also be used to optimize the performance of uprobes. 

Of course, it is already possible to patch user-space programs by directly modifying the text pages in memory. That is risky, however, because a misplaced patch can wreak havoc on a program. BPF might be useful here for the same reason it is in the kernel, Borkmann thought: BPF programs can be verified to ensure that they do not cause a crash. There are plenty of critical user-space applications where a form of live-patching that was guaranteed not to cause a crash would be welcome. 

Borkmann envisioned a system where the kernel would verify a BPF program, compile it to position-independent code, and then inject it into the executable memory of a running user-space application. The user-space process would run the compiled BPF program directly. One complication would be that the BPF program would not have access to kernel-only interfaces, such as the majority of BPF maps, but for simple patches access to BPF arenas should be sufficient. 

Song Liu thought that many important application properties could not be verified, and that therefore BPF programs in user-space might still crash user-space applications. Borkmann didn't think that small programs similar to Fastabend's "shields" would cause problems. In the kernel, the verifier has a clearly defined set of things that programs are and aren't allowed to do, Liu said, which is not true in user-space, so it's not clear that the verifier would be sufficient, or even helpful. 

Jakub Sitnicki asked what advantage a BPF-patching mechanism would have over uprobes. Borkmann said that, because BPF would run directly in user space, it would avoid the overhead of a trap into the kernel. Andrii Nakryiko agreed, noting that uprobes have limited performance. Liu suggested that some of uprobe's functionality could be moved into user-space without involving BPF. 

There is also the question of how to hook the generated code into the user-space executable, Nakryiko pointed out. Uprobes use a small breakpoint instruction, and then the kernel emulates the overwritten instruction. But patching in a longer jump to a BPF trampoline would be more complicated: ""this rewriting of generic instructions is very hard"". I asked about the possibility of including special target locations in user-space binaries that would be simpler to patch, but Nakryiko said that the benefit of uprobes was that they could run on any function. 

At that point, the session was running out of time, but not before one more audience member asked for the verifier to be pulled out into a separate library. A lot of the benefit of the verifier, especially type-system-related checks, carries over between contexts, he said; having the verifier as a separate library would let user-space applications add in their own rules on top of that. 

Overall, the assembled developers seemed to feel that using BPF for user-space patching would be difficult for several reasons, although none were opposed to the idea in principle, if it could be made to work. Whether anyone judges the ability to patch user-space programs in this way to be worth the hassle remains to be seen.
