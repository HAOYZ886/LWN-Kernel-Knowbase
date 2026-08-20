---
title: Direct host system calls from KVM
url: https://lwn.net/Articles/902585/
date: "July 29, 2022"
category: "Virtualization-KVM"
author: "By Jonathan Corbet July 29, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 29, 2022

As a general rule, virtualization mechanisms are designed to provide strong isolation between a host and the guest systems that it runs. The guests are not trusted, and their ability to access or influence anything outside of their virtual machines must be tightly controlled. So a patch series allowing guests to execute arbitrary system calls in the host context might be expected to be the cause of significantly elevated eyebrows across the net. Andrei Vagin has posted [such a series](<https://lwn.net/ml/linux-kernel/20220722230241.1944655-1-avagin@google.com/>) with the expected results. 

The use case for Vagin's work is [gVisor](<https://gvisor.dev/>), a container-management platform with a focus on security. Like a full virtualization system, gVisor runs containers within a virtual machine (using KVM), but the purpose is not to fully isolate those containers from the system. Instead, KVM is used to provide address-space isolation for processes within containers, but the resulting virtual machines do not run a normal operating-system kernel. Instead, they run a special gVisor kernel that handles system calls made by the contained processes, making security decisions as it goes. 

That kernel works in an interesting way; it maps itself into each virtual machine's address space to match its layout on the host, then switches between the two as needed. The function to go to the virtual-machine side is called, perhaps inevitably, `bluepill()`. The execution environment is essentially the same on either side, with the same memory layout, but the guest side is constrained by the boundaries placed on the virtual machine. 

Many of the application's system calls can be executed by gVisor within the virtual machine, but some of them must be handled in the less-constrained context of the host. It certainly works for gVisor to simply perform a virtual-machine exit to have the controlling process on the host side execute the call, then return the result back into the virtual machine, but exits are slow. Performing a lot of exits can badly hurt the performance of the workload overall; since part of the purpose of a system like gVisor is to provide better performance than pure virtualization, that is seen as undesirable. 

The proposed solution is to provide a new hypercall (`KVM_HC_HOST_SYSCALL`) that the guest kernel can use to run a system call directly on the host. It takes two parameters: the system-call number and a `pt_regs` structure containing the parameters for that system call. After executing the call in the host context (without actually exiting from the virtual machine), this hypercall will return the result back to the caller. This interface only works if the guest knows enough about the host's memory layout to provide sensible system-call parameters; in the gVisor case, where the memory layout is the same on both sides, no special attention is required. 

Internally, this functionality works by way of a new helper called [`do_ksyscall_64()`](<https://lwn.net/ml/linux-kernel/20220722230241.1944655-2-avagin@google.com/>), which can invoke any system call from within the kernel. Given that invoking system calls in this way is generally frowned upon, this functionality seems sure to be a lightning rod for criticism and, indeed, Thomas Gleixner duly [complained](<https://lwn.net/ml/linux-kernel/87a68vtvhf.ffs@tglx/>): ""this exposes a magic kernel syscall interface to random driver writers. Seriously no"". While he acknowledged that the series overall is ""a clever idea"", he made it clear that exposing system calls in this way was not going to fly. 

Meanwhile, the ability to invoke host-side system calls directly from a KVM guest pokes a major hole in the isolation between virtual machines and the host. Indeed, the cover letter describes it as ""a backdoor for regular virtual machines"". Thus, as one would expect, the direct system-call feature is disabled by default; processes that want to use it must enable it explicitly when creating a virtual machine. Most hypervisors, it is to be expected, will not do that. 

The kernels running deep within companies like Google often contain significant changes that are not found in the upstream code; this patch set gives a hint of what one of those changes looks like: 

> In the Google kernel, we have a kvm-like subsystem designed especially for gVisor. This change is the first step of integrating it into the KVM code base and making it available to all Linux users. 

That led Sean Christopherson to [ask](<https://lwn.net/ml/linux-kernel/Yts1tUfPxdPH5XGs@google.com/>) about what the following steps would be. ""It's practically impossible to review this series without first understanding the bigger picture"". Merging this first step could be a mistake if the following steps turn out not to be acceptable; at that point, the kernel community could find itself supporting a partial feature that is not actually being used. As it turns out, Vagin [said](<https://lwn.net/ml/linux-kernel/CAEWA0a4hrRb5HYLqa1Q47=guY6TLsWSJ_zxNjOXXV2jCjUekUA@mail.gmail.com/>), this is the only feature that is needed. gVisor works on top of KVM now, he said; the current patch series just improves its performance. 

Christopherson also asked about alternatives, noting that ""making arbitrary syscalls from within KVM is mildly terrifying"". Vagin provided a few, starting with the current scheme where a virtual-machine exit is used to (slowly) handle each system call. Another approach is to run _all_ of gVisor on the host side, exiting from the virtual machine for every system call. Executing a system call in this mode takes about 2.1µs; the direct system-call mechanism reduces that to about 1.0µs. Or gVisor could use BPF to handle the system calls; that provides similar performance, Vagin said, but would require some questionable changes, like providing BPF programs with the ability to invoke arbitrary system calls. Yet another possibility is to use the once-proposed [`process_vm_exec()` system call](<https://lwn.net/Articles/852662/>), but that can perform poorly in some situations. 

KVM maintainer Paolo Bonzini [said](<https://lwn.net/ml/linux-kernel/69b45487-ce0e-d643-6c48-03c5943ce2e6@redhat.com/>) that his largest objection is the lack of address translation between the guest and the host. In its current form, this mechanism depends on the memory layout being the same on both sides; otherwise any addresses in an argument to a system call would not make sense on the host side. As a result, the new mechanism is highly specialized for gVisor and seems unlikely to be more widely useful. It is not clear that everybody sees that specialization as a disadvantage, though. 

All told, gVisor in this mode represents an interesting shift in the security boundary between a host and the containers it runs. Much of the security depends on code that is within the virtual machine, with the host side trusting that code at a fairly deep level. It is a different view of how virtualization with KVM is meant to work, but it seems that the result works well — within Google at least. Whether this mechanism will make it into the mainline remains an open question, though. Making holes in the wall between host and guest is not something to be done lightly, so the developers involved are likely to want to be sure that no better alternatives exist.
