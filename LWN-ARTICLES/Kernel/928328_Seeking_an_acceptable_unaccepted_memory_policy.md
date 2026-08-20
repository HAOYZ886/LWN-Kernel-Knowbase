---
title: Seeking an acceptable unaccepted memory policy
url: https://lwn.net/Articles/928328/
date: "April 6, 2023"
category: "Architectures-x86; Confidential computing; Memory management; Releases-6.5"
author: "By Jonathan Corbet April 6, 2023"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 6, 2023

Operating systems have traditionally used all of the memory that the hardware provides to them. The advent of virtualization and confidential computing is changing this picture somewhat, though; the system can now be more picky about which memory it will use. Patches to add support for explicit memory acceptance when running under AMD's [Secure Encrypted Virtualization and Secure Nested Paging](<https://www.amd.com/en/developer/sev.html>) (SEV-SNP), though, have run into some turbulence over how to handle a backward-compatibility issue. 

Normally, when the operating-system kernel boots, it discovers the available memory and happily sets itself up to use that memory. Version 2.9 of the UEFI specification, though, added the concept of unaccepted memory; when this mechanism is in use, a system (normally a virtualized guest) will be launched with its memory in an unaccepted state. That system will not be able to make use of the memory provided until that memory has been explicitly accepted. On such systems, the bootloader will typically pre-accept enough memory to allow the guest kernel to boot; that kernel must take responsibility for accepting the rest before using it. 

Documentation on the motivation for this feature is scarce, but there would appear to be a couple of reasons for the addition of this new complication: 

  * Secure guest environments like SEV-SNP and Intel's TDX can protect their memory contents from the host and other guests through encryption, reverse-mapping tables, and more. Setting that protection up takes some time, though, slowing the boot process considerably. An explicit acceptance step allows the operating system to spread the initialization of memory over time. If memory is only accepted in chunks as it is needed, the system will boot into a running state more quickly. [The patches adding unaccepted-memory support](<https://lwn.net/ml/linux-kernel/20230330114956.20342-1-kirill.shutemov@linux.intel.com/>) from Kirill Shutemov take advantage of this by deferring acceptance of memory until it is needed. 
  * Explicit acceptance can help to defend a secure guest from a malicious hypervisor that might try to play games with the guest's memory behind the scenes. Should the hypervisor try to sneak a new page into a guest's address space, that new memory will not have been accepted by the guest and an attempt to access it will generate a fault. 

Each vendor's secure environment has its own way of managing the acceptance process, so some of the code that implements acceptance must necessarily be specific to one subarchitecture. Shutemov's patches add support for Intel's TDX, but support for AMD's SEV-SNP comes from [a separate patch set](<https://lwn.net/ml/linux-kernel/cover.1680628986.git.thomas.lendacky@amd.com/>) from Tom Lendacky. 

It turns out that SEV-SNP support has to handle a problem that TDX does not: existing users. The kernel has been able to work with SEV-SNP since the 5.19 release, so there are already systems using SEV-SNP in the wild. But current kernels, while they understand SEV-SNP, do not have support for memory acceptance, or even the concept that memory must be explicitly accepted. If such a kernel is booted on a system where some of the memory has not been accepted, it will be unable to use that memory and may fail badly trying. 

That is not the security experience that SEV-SNP was created to provide. To avoid such an outcome, Lendacky's series includes [a patch](<https://lwn.net/ml/linux-kernel/1d38d28c2731075d66ac65b56b813a138900f638.1680628986.git.thomas.lendacky@amd.com/>) from Dionna Glaze adding a special UEFI protocol to provide compatibility for older systems. Specifically, when running on AMD hardware, a booting system must invoke the new UEFI protocol prior to the call to [`ExitBootServices()`](<https://edk2-docs.gitbook.io/edk-ii-uefi-driver-writer-s-guide/5_uefi_services/readme.3/5312_exitbootservices>) that transfers full control away from the firmware. If the call to the new protocol is _not_ made, the firmware will pre-accept all of the memory provided to the system before handing control to the operating system. 

This mechanism lets kernels that are capable of handling unaccepted memory inform the firmware of that fact while avoiding problems for kernels that lack that ability. The plan is that this will be a temporary measure, only needed until users can be expected to have newer kernels: 

> This protocol will be removed after the end of life of the first LTS that includes it, in order to give firmware implementations an expiration date for it. When the protocol is removed, firmware will strictly infer that a SEV-SNP VM is running an OS that supports the unaccepted memory type. 

When [an earlier version](<https://lwn.net/ml/linux-kernel/20230113212926.2904735-1-dionnaglaze@google.com/>) of this patch was posted in January, Shutemov [objected](<https://lwn.net/ml/linux-kernel/20230113222024.rp2erl54vx3grdbd@box.shutemov.name/>), calling the feature ""a bad idea"". He [added](<https://lwn.net/ml/linux-kernel/20230116134246.soworigs56bz5v7o@box.shutemov.name/>): ""This patch adds complexity, breaks what works and the only upside will turn into a dead weight soon"". X86 maintainer Dave Hansen [agreed](<https://lwn.net/ml/linux-kernel/def9b0b5-b880-be99-fa95-b05d76a91824@intel.com/>), worrying that it would never be possible to remove support for this interface once it had been added. 

Shutemov [reiterated](<https://lwn.net/ml/linux-kernel/20230404174506.pjdikxvk2fsyy4au@box.shutemov.name/>) his opposition in response to the most recent patch set, but this time Hansen [indicated](<https://lwn.net/ml/linux-kernel/bc9e6d82-c7c1-47dc-e91f-57d9b4e2bb0a@intel.com/>) that he had changed his mind: 

> The fact is that we have upstream kernels out there with SEV-SNP support that don't know anything about unaccepted memory. They're either relegated to using the pre-accepted memory (4GB??) or _some_ entity needs to accept the memory. That entity obviously can't be the kernel unless we backport unaccepted memory support. 

He would like to pretend that the problem doesn't exist, he continued, but ""my powers of self-delusion do have their limits"". 

Shutemov [was unswayed](<https://lwn.net/ml/linux-kernel/20230404180917.4fsgkzcdhqvph6io@box.shutemov.name/>), though, suggesting that the hypervisor could load a special firmware that pre-accepts all of the memory when launching a system that lacks that support; how the hypervisor would know that about any specific guest is not entirely clear. Ard Biesheuvel [disagreed](<https://lwn.net/ml/linux-kernel/CAMj1kXF0XyEOuSUDqgsLSYK8GSkGN1xK3RQ525+BxhG+7+vnCA@mail.gmail.com/>), arguing that letting the kernel make its own capabilities known to the firmware is the most straightforward solution to a problem that cannot be ignored. When Shutemov [said](<https://lwn.net/ml/linux-kernel/20230405134226.2ptk3rd7mfin3wlm@box.shutemov.name/>) that this protocol would fail in cases where the bootloader calls `ExitBootServices()` prior to starting the kernel, Biesheuvel [answered](<https://lwn.net/ml/linux-kernel/CAMj1kXGgvk1=QQOJ_u-VqTyoOe68zv2cJYxuJfsvJR4-r5QshA@mail.gmail.com/>) that it was ""a theoretical concern"" that will not show up in real-world use. 

After holding a finger up to the wind, your editor's guess is that this feature will eventually be accepted into the mainline. A more interesting question, perhaps, is when it will be removed. Biesheuvel [said](<https://lwn.net/ml/linux-kernel/CAMj1kXEDYa5YcRj7t9JJmQ21+ierue2q6ADOXUOMjoOGndGRug@mail.gmail.com/>) that, over time, firmware will stop supporting this protocol, and it will be possible to remove that support from the kernel as well. Hansen [is unconvinced](<https://lwn.net/ml/linux-kernel/e46777d3-adea-90a2-afc4-35f9d7cef50c@intel.com/>), though; he notes that users run old kernels for a long time, so the support for them will also need to stay for a long time. Or, as he [said](<https://lwn.net/ml/linux-kernel/d3d95889-67ec-576e-ec9c-b9bd462e1c0e@intel.com/>) earlier in the discussion: ""Yeah, the only real expiration date for an ABI is "never". I don't believe for a second that we'll ever be able to remove the interface."" 

There is nothing unusual about this situation; whenever maintaining compatibility is a concern, software will fill up with little hacks like this. That is part of the cost of keeping things working. In this case, the cost appears small enough to be acceptable. Existing SEV-SNP users will, once this work is merged, be able to run their virtual machines on systems where memory must be explicitly accepted prior to use.
