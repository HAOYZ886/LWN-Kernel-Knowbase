---
title: Enhancing KVM for guest protection and security
url: https://lwn.net/Articles/805114/
date: "November 20, 2019"
category: KVM
author: "November 20, 2019 This article was contributed by Sergio Lopez KVM Forum"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

November 20, 2019

This article was contributed by Sergio Lopez

* * *

[KVM Forum](<https://lwn.net/Archives/ConferenceByYear/#2019-KVM_Forum>)

A key tenet in KVM is to reuse as much Linux infrastructure as possible and focus specifically on processor virtualization. Back in 2007, this meant a smaller code base and less friction with the other kernel subsystems, especially when compared with other virtualization technologies such as Xen. This led to KVM being merged into the mainline with relative ease.

But now, in the era of microarchitectural vulnerabilities, the priorities have shifted, and the KVM's reliance on other kernel subsystems can be a liability. For one thing, the host kernel widens the [TCB (Trusted Computing Base)](<https://en.wikipedia.org/wiki/Trusted_computing_base>) and makes for a larger attack surface. In addition, kernel data structures such as the direct memory map give Linux access to guest memory even when it is not strictly necessary and make it impossible to fully enforce the principle of least privilege. In his talk "Enhancing KVM for Guest Protection and Security" ([slides [PDF]](<https://static.sched.com/hosted_files/kvmforum2019/23/nakajima-enhancing-kvm-for-guest-protection.pdf>)) presented at [KVM Forum 2019](<https://events19.linuxfoundation.org/events/kvm-forum-2019/>), long-time KVM contributor Jun Nakajima explained this risk and suggested some strategies to mitigate it.

#### Removing the VMM from the TCB

Nakajima pointed to three main security issues in the current KVM hypervisor design: piggybacking on Linux subsystems, the user-space [virtual machine monitor](<https://en.wikipedia.org/wiki/Hypervisor>) (VMM) having access to the data of the guest it is managing, and the kernel having access to all address spaces and data structures, including those from every guest running in the same host. Inspired by virtualization-oriented memory encryption technologies, like [AMD SEV](<https://developer.amd.com/sev/>), he proposes a security strategy based on removing as many elements as possible from the TCB, as seen from the guest perspective.

The first step would be removing the user-space VMM from the TCB. To achieve this, the guest would use a new KVM facility to specify a virtual memory region to be shared, while the rest of the memory would be marked as private to the guest. Any attempts from the VMM to access a memory region that the guest doesn't want to be shared will result in a page fault and, depending on the implementation, potentially send a signal to the VMM.

This implies that the kernel running inside the guest needs be modified to make use of the facility and to ensure that DMA operations work exclusively within the range of the memory region shared with the VMM, which can be done using a software I/O translation buffer (an swiotlb-based bounce buffer). This strategy is already being used for supporting AMD SEV, which allows guests to run with memory encryption (making it inaccessible from either the host's kernel or the user-space VMM), and Nakajima pointed out that the intention is to rely on the existing code as much as possible.

#### Protecting guests from the host kernel

Removing the user-space VMM from the TCB is useful, but it is just the first step. Going further, and removing the host Linux kernel from the TCB, requires deeper changes so that the hypervisor can "absorb" the host kernel and deprive it of its normal full privileges on the machine.

This feat had already been presented ([slides [PDF]](<http://www.linux-kvm.org/images/4/40/01x05-Jun_Nakajima-Kernel_Protection_Using_Hardware-Based_Virtualization.pdf>)) at KVM Forum 2016 and, in the meantime, got the name virtualization-based hardening (VBH). As that name suggests, the hypervisor protects itself from the host kernel through hardware virtualization. Once the hypervisor has initialized itself, Linux would run in guest mode (with special privileges) and the hypervisor could use [extended and nested page tables](<https://en.wikipedia.org/wiki/Second_Level_Address_Translation#Rapid_Virtualization_Indexing>) (EPT/NPT) along with an IOMMU to control Linux's access to physical memory.

With this execution model, KVM would be able to provide virtual memory contexts to guests without relying on the Linux memory-management subsystem, which would be still in use for servicing both the user-space processes running in the host and the kernel itself. This way, KVM will be the only subsystem able to access address space mappings and alter their properties (including their protection levels), so even if an attacker happens to take control of some other kernel subsystem, the guest's memory won't be compromised.

In this scenario, while still far from being an actual [Type-1 Hypervisor](<https://en.wikipedia.org/wiki/Hypervisor#Classification>), KVM would effectively hijack a significant part of the low-level virtual memory functionality from Linux. Of course, this means that the hypervisor would need to provide its own mechanism for swapping pages, and define new interfaces to be able to access the actual memory backends (RAM, software-defined, NVDIMM, etc.).

This strategy shares some similarities with one also presented at KVM Forum 2019 by Liran Alon and Alexandre Chartre in their talk "KVM ASI (Address Space Isolation)" ([slides [PDF]](<https://static.sched.com/hosted_files/kvmforum2019/34/KVM%20Forum%202019%20KVM%20ASI.pdf>)). In it, they suggested creating a virtual address space for each guest that would exclusively map per-VM data, KVM, and the needed kernel subsystems. This is less radical in the sense that it would still be using the Linux memory-management facilities, and thus probably be easier to get accepted upstream, but at the cost of keeping a larger TCB.

All in all, it seems like a consensus is being built around the idea that it's necessary to rethink the way in which KVM manages the guest's memory, and its relationship with the rest of the Linux subsystems.

#### Going further: removing KVM from the TCB

So far, the strategies presented were able to remove the VMM and the host kernel from the TCB. The last step would be removing KVM itself from the TCB. For this to be possible, KVM must remove guest memory from its address space, including from the Linux direct map. Only regions explicitly shared by the guests would be accessible to KVM. This is problematic because currently some operations serviced during VM-exits assume that KVM has full access to whole guest memory. VM-exits are performed when the VM returns control to the host's kernel to handle an operation it cannot perform.

To overcome this issue, Nakajima proposes two options. One is to adapt the kernel running inside the guest. Operations that would trigger VM-exits, and for which KVM would need to to access the guest's memory, would be replaced with explicit hypercalls. Alternatively, in order to run unmodified drivers, the operations could be reflected to a virtualization exception (#VE) handler. The handler would emulate the memory-mapped I/O (MMIO) operations and translate them to hypercalls.

Either of these strategies can be used to enable isolation of guest memory from the hypervisor. This would be helpful to prevent accidental leaks and side-channel attacks, but an attacker gaining full control of KVM would still be able to alter mappings and, potentially, gain access to the private regions of every guest. A complete mitigation of this risk therefore requires hardware assistance, like the one provided by AMD SEV and its successors, [SEV-ES (Encrypted State) [PDF]](<https://www.amd.com/system/files/TechDocs/Protecting%20VM%20Register%20State%20with%20SEV-ES.pdf>) and [SEV-SNP (Secure Nested Paging) [PDF slides]](<https://static.sched.com/hosted_files/lsseu2019/65/SEV-SNP%20Slides%20Nov%201%202019.pdf>). This allows the guest memory to be transparently encrypted in such a way that not even the hypervisor is able to access it in the clear.

#### Proof of concept and performance

Lastly, Nakajima presented a proof of concept that implements the ability to remove the guest mappings from both the VMM and the host's kernel, and gave some initial numbers about the performance impact. For disk read operations on a virtio device, he measured a 1.2% increase in CPU time with one guest, and 1.3% with more than ten. For write operations, the impact was slightly lower, with 1.1% for a single guest and 1.2% for the more than ten guests case. For network send operations, also on a virtio device, the increase in CPU time was of 2.6% when running a single guest, and 3.8% with more than ten.

According to Nakajima, the next step is finishing the proof of concept, and then sharing the patches with the upstream community. This will probably spark an interesting discussion in the upstream mailing lists about this and, possibly, other address space isolation techniques for KVM to protect the guests' memory from both an attacker controlling the host and [side-channel](<https://en.wikipedia.org/wiki/Side-channel_attack>) leaks.
