---
title: KVM for Android
url: https://lwn.net/Articles/836693/
date: "November 11, 2020"
category: Android; KVM
author: "By Jake Edge November 11, 2020 KVM Forum"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
November 11, 2020

* * *

[KVM Forum](<https://lwn.net/Archives/ConferenceByYear/#2020-KVM_Forum>)

A Google project aims to bring the Linux kernel virtualization mechanism, KVM, to Android systems. Will Deacon leads that effort and he (virtually) came to [KVM Forum](<https://events.linuxfoundation.org/kvm-forum/>) to discuss the project, its goals, and some of the challenges it has faced. Unlike some Android projects of the past, though, "protected KVM" is being worked on in the open, with code going upstream along the way. 

Deacon is one of the maintainers of the arm64 architecture for the kernel, as well as a maintainer and contributor in various other parts of the kernel, including concurrency, locking, atomic operations, and tools for the kernel memory model. He has worked in the kernel for a long time, but not really on KVM; the closest he had come to that is maintaining the Arm IOMMU drivers. He started working on the Android Systems team at Google in 2019 "and found myself leading the protected KVM project", which is the KVM on Android effort. 

The project is the top contributor to KVM for arm64 for the 5.9 and 5.10 kernels; KVM seems to be a "hot topic" right now, he said, and not just for arm64, but for other architectures as well. All of the project's work is being upstreamed as it goes, so what he was presenting was "very much a work in progress". He wants to avoid the trap of doing a bunch of work out of tree and then "throwing it over the wall", which does not lead to good solutions that are embraced by the community. 

#### Android background

The latest development for the overall Android system is the [generic kernel image](<https://source.android.com/devices/architecture/kernel/generic-kernel-image>) (GKI), which is meant to [reduce Android kernel fragmentation](<https://lwn.net/Articles/830979/>). Traditionally, each handset had its own kernel version, which simply does not scale. That leads to fragmentation, which in turn leads to the inability to update some systems because of the difficulties and expenses associated with updating multiple kernel versions, one for each different device. It can also make it impossible to update the Android release on certain devices because their kernel is too old to have a feature needed by the more recent Android release. 

One other problem that stems from this fragmentation does not get enough attention, he said: it is also bad for the upstream kernel. The idea behind the mainline kernel is to have the right subsystems and abstractions to be able to support a wide variety of hardware, but that cannot be done unless the developers have visibility into all of the different problems and solutions for all of the disparate hardware. Because the code is all "squirreled away in all these different kernels, it's very hard to see the wood for the trees"; that means the kernel developers cannot come up with an abstraction that will work for everyone. 

GKI is meant to solve that problem by "rallying around" a given kernel version that is tied to a particular Android release. A limited subset of the module ABI will be maintained as a stable interface for that kernel. Vendors can then create driver modules that will continue to work as the kernel gets long-term support (LTS) and security updates. 

With a grin, Deacon said that he could hear audience members strongly suggesting ("screaming") that the Android systems team _not_ maintain the ABI as the kernel evolves. He acknowledged the problems with that, but noted that the team has identified a strict subset of the symbols in the ABI that it will continue to maintain—only for a single kernel version and Android release pair. 

#### Android virtualization today

The hypervisor situation on Android is chaotic. "If you think fragmentation on the kernel side is bad, this is much, much worse." At least all of the Android devices are running some version of Linux, but in terms of hypervisors, "it's the wild west of fragmentation". Some devices do not have a hypervisor at all, which simplifies the picture, but many do, and they are used for several different things. 

The first main use is for security enhancements that are meant to protect the kernel but are sometimes problematic in their own right. He pointed to Jann Horn's [Project Zero blog post](<https://googleprojectzero.blogspot.com/2020/02/mitigations-are-attack-surface-too.html>) that notes: ""Mitigations are attack surface, too"". It shows how attacks can be made against some of these security enhancements. It is important to remember that the hypervisor is running with elevated privileges, so bugs there can mean that these supposed protections are not really protecting the system. 

Another hypervisor use in Android today is for coarse-grained memory partitioning that looks something like an IOMMU but actually is not. It is used at boot time to carve up the physical memory into regions that can be handed off to various devices for DMA and other uses. He understands why that is needed, but there is a lot more that could be done with a hypervisor after boot time, so this type of use is kind of a waste, he said. 

The final reason that hypervisors are used in Android today is his least favorite: running code outside of Android itself. Armv8 has multiple privilege levels, called [exception levels](<https://developer.arm.com/architectures/learn-the-architecture/exception-model/privilege-and-exception-levels>), going from the most privileged, firmware (EL3), through the hypervisor (EL2) and operating system (EL1) levels, to the least privileged user (EL0) level. The hypervisor exception level is not the firmware level, so device makers do not have to worry about bricking devices when updating code there, and it is not the operating system level, so code running there does not need to integrate with anything else. That means EL2 has become something of a "playground"; code that doesn't seem to fit anywhere else gets stuck there, which is bad because EL2 has lots more privileges than are probably needed. 

In most cases, there are not even any virtual machines (VMs), so these hypervisors are not providing the usual services. His conclusion is that both security and functionality are losing out because of that. Security is hampered because there is an increased [trusted computing base](<https://en.wikipedia.org/wiki/Trusted_computing_base>) (TCB) and it is more difficult to update the devices because of the fragmentation at that level. And functionality is lacking because there is no access to the hardware virtualization features from within Android. 

He then described the Armv8 "exception model", showing how the various levels of software are built up from most to least privileged, but also how Arm has long had a parallel "trusted" side where applications can be run on a trusted OS and hypervisor. The definition of "trusted" is just a "bit on the bus" that allows more access to physical memory. It is important to note that code on the trusted side can access all of the memory, while code on the untrusted side is unable to access trusted-only memory. 

Effectively, the trusted levels are all more privileged than the non-secure levels, so the trusted OS can map non-trusted hypervisor memory, for example, and it could provide access so that trusted applications have access to it too. That is problematic in the Android world in part because of what is typically running on the trusted side: third-party code for digital rights management (DRM), various opaque binary blobs, cryptographic code, and so on. That code may not be trustworthy and it suffers from the fragmentation problem as well. What people think of as "Android" is running in the least-privileged part of the system. 

The term "trusted" is largely a marketing term, he thinks, to make people feel that the code running there is safe and reliable. But there is another definition of "trust", to "expect, hope, or suppose", and that is also operative here. The Android system has to hope that the software running in the trusted side is not malicious or compromised because there is not anything Android can do if it is. 

Instead, the Android project would like to have a way to de-privilege this third-party code. There is a need for a portable environment that can host these services in a way that is isolated from the Android system. That mechanism would also isolate these third-party programs from each other. 

#### Enter KVM

One way to do that is to move the trusted code into a VM at the same level as the Android system. The third-party code would be no more (or less) trusted than Android itself. Since there are no VMs currently in Android, there is an opening to add some if there is a hypervisor available to manage them. The idea is to use the GKI effort to introduce KVM as that hypervisor in order to move that third-party code out of the over-privileged trusted region. 

All arm64 Android devices support virtualization in hardware and have two-stage MMUs, which allow partitioning the memory so that guests cannot access outside of their memory regions. KVM has been supported on arm64 since Linux 3.11 (in 2013). There are two basic modes that are supported depending on whether the [Virtualization Host Extensions](<https://developer.arm.com/architectures/learn-the-architecture/aarch64-virtualization/virtualization-host-extensions>) (VHE) support is available; that support was added to v8.1 of the architecture, but all arm64 processors can still run in the earlier non-VHE (nVHE) mode if they choose. 

In nVHE mode, the host and guest kernels both run at the operating system level (EL1), while there is a virtual machine monitor (VMM) at the EL2 hypervisor level. Because the host kernel does not have the privileges needed to directly switch to and from the guests, the VMM must do a "world switch" to make that happen, which makes nVHE mode relatively slow. [**Update** : The article confuses the VMM and world-switch code, which Deacon helpfully untangles in a comment below.] 

In v8.1, the VHE support allowed EL2 programs to have fewer constraints, so the host kernel could be run in EL2, with all of the guests as VMs in EL1, which is "blazingly fast". That mode is not really compatible with the threat model for Android, however. It moves the host kernel and VMM (via `ioctl()`) into the TCB and the host kernel has access to all of the memory of the guests. It effectively turns the trusted model on its head, so only the Android system would be in a privileged position, which is not desirable either. 

The envisioned Android security model requires that guest data remains private even if the host kernel is compromised, and KVM using VHE does not work that way. But that is not a problem with nVHE mode, so it might make sense to revisit that. Instead of trusting the full host kernel, only the world-switch piece needs to be trusted. It can be extended to manage the stage-2 page tables and manage other functions for the guests. Message passing can be used between the host kernel and the VMs and a special bootloader can be used to ensure that the host does not tamper with the VM images. "While we're at it, we'll try to apply formal verification techniques because the EL2 code is drastically simpler than Linux." 

Another possibility would be to run Android in a VM, which is plausible, but Deacon does not really think that is demonstrably better; it has a different set of challenges. Interrupt latency could be a problem for an Android VM. There is also a need for device pass-through and he does not think the Arm IOMMUs are really up to handling that at this point. 

The nVHE execution environment in EL2 is "a pretty horrible place"; it has its own limited virtual address space that lacks the addressing capability needed for running general kernel code. Any code running there is not preemptible or interruptible, so you cannot block or schedule. EL2 can access all of the memory if it is mapped, but, because of that, the project does not want to put a lot of complicated code there—that would defeat the purpose. There is "very limited device access at EL2" because the host kernel normally handles all of that; typically, there is no console at EL2, though they have some hacks for a debug console. 

The EL2 code needs to be self-contained and safe against a compromised host, which is not the case for kernels prior to 5.9, where KVM could effectively cause arbitrary code to be run as an EL2 hypercall by passing a function pointer for what to call. As part of the recent changes, the project has switched to a fixed set of hypercalls for the services that need to be provided. The EL2 payload is embedded in a separate ELF section that uses symbol prefixing to ensure that symbols from the host kernel are not wrongly used. As the system boots, the host kernel sets the static keys appropriately before it de-privileges itself by moving to EL1; the EL2 object is then no longer mapped from EL1, so those changes are one-way. 

#### Open problems

There are still a number of problems, however, most of which come down to how the virtual memory is managed. Today, the host kernel is in control of the hypervisor's virtual memory, which is obviously a problem given the Android use case. The stage-1 mappings are created by the host kernel, which means it can change the page table out from under the hypervisor; it can also write to any of the hypervisor memory. 

Beyond that, the stage-2 page tables for guests are also managed by the host kernel. When EL2 does a world switch, it just blindly installs those tables assuming that the host kernel is doing the right thing. That obviously needs to change as well. The protected KVM project has some patches that it is targeting for Linux 5.11 to change page-table handling, some of which (e.g. [page table and fault handling](<https://lore.kernel.org/kvmarm/20200911132529.19844-1-will@kernel.org/>), [per-CPU data handling](<https://lore.kernel.org/lkml/20200922204910.7265-1-dbrazdil@google.com/>)) have already landed in 5.10. 

Moving the page-table handling code to EL2 has some interesting properties. When a new guest is created, its memory will be unmapped from the host, which is not something that Linux can deal with. It is not like memory hotplug, where a whole bank can go away, it is just the pages assigned to the guest that will disappear. The [KVM protected memory extension](<https://lwn.net/Articles/835342/>) patches would fix that problem, though they have not been merged. They would allow handling the case where guest memory disappears and then reappears later when the guest is torn down. 

IOMMU support is needed to avoid DMA attacks, but the current systems-on-chip (SoCs) are not really ready for that. Ideally, the IOMMUs would simply reuse the page tables that are already installed in the CPU, so there would be limited IOMMU-management code needed in EL2. It would not be desirable to have multiple different IOMMU drivers bloating the EL2 code. 

Another piece that will be needed is the template bootloader that is used to start guests; it will be "very very small" and the current plan is to write it in bare-metal Rust. It will check the signature of the VM image to ensure that it has not been tampered with; if it passes, the bootloader will jump to it. That image will have a "proper second-stage bootloader" as part of it, so the template bootloader can remain extremely simple. None of that is particularly arm64-specific, so other architectures may be able to use it as well. 

#### Virtual platform

The protected KVM project is adapting the [Chrome OS VMM](<https://chromium.googlesource.com/chromiumos/platform/crosvm/>) (crosvm) for its VMM. Crosvm is now included in the Android open-source project (AOSP); there have been lots of talks about it at KVM Forums, he said. Crosvm is written in Rust with a major focus on security and sandboxing, which makes it a good match. It also has many virtio devices already implemented and is cross-architecture, which is important, perhaps surprisingly, in part because of the [Cuttlefish](<https://source.android.com/setup/create/cuttlefish>) virtual Android device that is based on the x86 architecture. 

Protected KVM provides a fairly basic arm64 virtual platform for guests, with much of what would be expected. One major difference is that it provides the [Reduced Virtual Interrupt Controller [PDF]](<https://developer.arm.com/architectures/system-architectures/software-standards/rvic>) (RVIC, which is a paravirtual IC), rather than the standard [Generic Interrupt Controller](<https://developer.arm.com/ip-products/system-ip/system-controllers/interrupt-controllers>) (GIC) because the latter is more complicated than what the developers want to add to the EL2 code. 

For I/O, the obvious answer is virtio, he said, but that does not fully solve the problem because it assumes that hosts have access to all of guest memory. Even if you work around that, it means that the host can intercept the I/O data for the guest, which means "you have to use quite clever crypto". There is also no shared-memory device for virtio, so bounce buffers are needed. Support for that is working but has various undesirable properties, including slower performance, so some other solution is sought. 

His final slide was a long list of things that still need to be done. One big lurking item is something where he underestimated the effort required: getting it all working with the rest of the Android system. There is a lot needed to integrate with the user-space bits of the system, which his experience as a kernel (and now KVM) developer did not prepare him for. He encouraged those interested to contact the team or to post to the [KVM/Arm mailing list](<https://lists.cs.columbia.edu/mailman/listinfo/kvmarm>). The [PDF slides](<https://mirrors.edge.kernel.org/pub/linux/kernel/people/will/slides/kvmforum-2020-edited.pdf>) are available and the video can be [accessed](<https://lwn.net/Articles/836505/>) from the [event site](<https://www.accelevents.com/e/OSSELCEU2020>) and will eventually appear on YouTube.
