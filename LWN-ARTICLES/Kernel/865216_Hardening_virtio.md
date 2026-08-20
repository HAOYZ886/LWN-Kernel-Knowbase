---
title: Hardening virtio
url: https://lwn.net/Articles/865216/
date: "August 9, 2021"
category: "Virtualization-virtio"
author: "August 9, 2021 This article was contributed by Marta Rybczyńska"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

August 9, 2021

This article was contributed by Marta Rybczyńska

Traditionally, in virtualized environments, the host is trusted by its guests, and must protect itself from potentially malicious guests. With initiatives like confidential computing, this rule is extended in the other direction: the guest no longer trusts the host. This change of paradigm requires adding boundary defenses in places where there have been none before. Recently, Andi Kleen submitted a [patch set](<https://lwn.net/ml/linux-kernel/20210603004133.4079390-1-ak@linux.intel.com/>) attempting to add the needed protections in [virtio](<https://wiki.libvirt.org/page/Virtio>). The discussion that resulted from this patch set highlighted the need to secure virtio for a wider range of use cases.

Virtio offers a standardized interface for a number of device types (such as network or block devices). With virtio, the guest runs a simplified, common driver, and the host handles the connection to the real underlying device. The communication between the virtio device (host side) and the driver (guest side) happens using data structures called virtqueues, which are typically memory buffers, though the actual implementation depends on the bus used.

#### The scope of the hardening

In the confidential-computing world, the host is not allowed to access guest memory that was not explicitly shared with it. In addition, the guest's memory can be encrypted by the processor with a key unknown to the host. Kleen's work is designed to build on Intel's upcoming hardware feature, called [Trust Domain Extensions](<https://software.intel.com/content/www/us/en/develop/articles/intel-trust-domain-extensions.html>) (TDX), which is designed to protect guests in cloud environments. It is built using a number of architecture extensions, including memory encryption with [Multi-Key Total Memory Encryption](<https://software.intel.com/sites/default/files/managed/a5/16/Multi-Key-Total-Memory-Encryption-Spec.pdf>) (MKTME) (covered here when a different [memory-encryption API](<https://lwn.net/Articles/776688/>) was proposed in 2019), and a new CPU mode called Secure-Arbitration Mode (SEAM). In the protected mode, code running under SEAM can only use a specified (encrypted) memory range, while all other processes (and DMA operations) cannot access that zone. Virtio, as a commonly used interface between the guest and the host, must take extra care to avoid compromising the security that TDX provides.

Until recently, virtio drivers assumed that the other side could be trusted. As a consequence, they have sometimes lacked necessary checks when working with the various metadata (operation descriptors, ring positions, result codes, etc.) shared with the device (i.e. the host); thus they could fail to catch bad pointers, out-of-range buffer indices, and similar errors. A malicious host could thus exploit buffer overruns and gain access to guest memory. Checking metadata from devices is also necessary in other cases, as virtio is no longer only used between a guest and a host -- [some physical devices are now implementing the virtio interface](<https://lwn.net/Articles/805235/>).

The patches can be grouped into three parts. The first one is the hardening in virtio itself, placed in virtio-ring. It also includes the disabling of some virtio modes. The second part enables the mode restrictions for x86 systems with TDX enabled. Finally, the last part includes changes in swiotlb, which enables DMA operations in situations where they are not otherwise possible by copying data through an intermediate ("bounce") buffer. The hardening included in the patch set adds additional checks for malicious pointers.

#### Virtio modes

Virtio defines many modes with different memory organizations, depending on the needs of the device and the driver. This creates multiple code paths to harden; apparently some of them are easier to fix than the others. Kleen [decided](<https://lwn.net/ml/linux-kernel/20210603004133.4079390-2-ak@linux.intel.com/>) to protect only the so-called [split mode](<https://docs.oasis-open.org/virtio/virtio/v1.1/cs01/virtio-v1.1-cs01.html#x1-240006>), where each virtqueue consists of different parts, each of those writable by either the driver or the device, but not by both at the same time.

In the proposed patch set, the other modes are disallowed when the guest runs in the TDX protected mode. This choice disallows indirect descriptors, a split-mode extension that allows the allocation of a number of descriptors in a separate memory area, improving performance by increasing the capacity of the ring. Also disabled is the [packed mode](<https://docs.oasis-open.org/virtio/virtio/v1.1/cs01/virtio-v1.1-cs01.html#x1-610007>), a more compact, in-memory layout. This restriction caused a number of comments. Jason Wang [observed](<https://lwn.net/ml/linux-kernel/28c8302b-6833-10b4-c0eb-67456e7c4069@redhat.com/>) that disabling indirect descriptors causes a significant performance loss. Kleen [had problems](<https://lwn.net/ml/linux-kernel/5a2d0d70-fa6b-f08d-f222-5c00cf5f9d44@linux.intel.com/>) securing this mode and thinks it is too difficult to protect. Wang [thinks the problem can be solved](<https://lwn.net/ml/linux-kernel/93f58cfd-be59-1ea6-e0d0-6840c14d6980@redhat.com/>) and promised to post a patch set.

Andy Lutomirski also [disagreed](<https://lwn.net/ml/linux-kernel/cc5c8265-83f7-aeb1-bc30-3367fe68bc97@kernel.org/>) with the approach of disabling all modes except one. He [highlighted](<https://lwn.net/ml/linux-kernel/308e7187-1ea7-49a7-1083-84cf8654f52a@kernel.org/>), later in the thread, that devices must not be allowed to corrupt the driver in any setting, so the hardening should be more generic: 

> For most Linux drivers, a report that a misbehaving device can corrupt host memory is a bug, not a feature. If a USB device can corrupt kernel memory, that's a serious bug. If a USB-C device can corrupt kernel memory, that's also a serious bug, although, sadly, we probably have lots of these bugs. If a Firewire device can corrupt kernel memory, news at 11. If a Bluetooth or WiFi peer can corrupt kernel memory, people write sonnets about it and give it clever names. Why is virtio special? 

According to Lutomirski, the driver should be made secure for all use cases, not just the ones using TDX. Disabling other modes only when running TDX does not solve the problem, as bugs in those modes could be exploited to attack systems today. He also noted that virtio is not only implemented in software, but there are also hardware devices that expose a virtio-compatible interface. In [another message](<https://lwn.net/ml/linux-kernel/2b2dec75-a0c1-4013-ac49-a49f30d5ac3c@www.fastmail.com/>) he suggested splitting the driver into a modern version and a legacy one (including all modes that are not used in practice, or could not be fixed without compatibility issues) and actually harden the modern one completely.

Kleen [disagreed](<https://lwn.net/ml/linux-kernel/d3bf637e-556a-be48-39f9-dc7defd19092@linux.intel.com/>), stating that there is no memory protection in other cases (possibly those not using a mechanism like TDX) and there is a risk of compatibility problems (that he did not identify). The boundary checks are enabled unconditionally, but the other virtio modes are only disabled when TDX is active. The discussion ended this way, without clear conclusions.

#### Similar work

In the discussion, Wang [noted](<https://lwn.net/ml/linux-kernel/63d01084-68d2-a8d5-931d-541a22b5f231@redhat.com/>) that there are similar hardening needs, including support for [AMD Secure Encrypted Virtualization](<https://developer.amd.com/sev/>) (SEV). Another need for virtio hardening comes from SmartNICs and devices implementing virtio, notably including [vDPA](<https://lwn.net/Articles/816063/>) — a device type that implements virtio for the data path, but has a vendor driver for the control path — and [VDUSE](<https://lwn.net/ml/linux-kernel/20210713084656.232-1-xieyongji@bytedance.com/>), a vDPA device implemented in user space. They have similar problems and should not trust the metadata provided by the device. According to Kleen, those other cases [should work](<https://lwn.net/ml/linux-kernel/33504152-624a-45cc-51b3-10ce7aa2428f@linux.intel.com/>) with his changes with a few additions. 

#### Conclusions and next steps

Hardening device drivers against malicious devices is an objective welcomed by kernel developers. The discussion shows that there is a need, with multiple use cases, and that different pieces have fixes in the works. Kleen's patch set received mixed reviews in its current form. The main issue seems to be the fact that it is too closely linked to the TDX work and the kernel developers would prefer a more generic solution. We are likely going to see more iterations of this work, and other hardening fixes in virtio, in the future.
