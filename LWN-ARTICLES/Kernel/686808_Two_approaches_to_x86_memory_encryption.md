---
title: Two approaches to x86 memory encryption
url: https://lwn.net/Articles/686808/
date: "May 11, 2016"
category: "Security-Memory encryption"
author: "By Jonathan Corbet May 11, 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 11, 2016

Techniques for hardening the security of running systems often focus on access to memory. An attacker who can write (or even read) arbitrary memory regions will be able to take over the system in short order; even the ability to access small regions of memory can often be exploited. One possible defensive technique would be to encrypt the contents of memory so that an attacker can do nothing useful with it, even if access is somehow gained; this type of encryption clearly requires hardware support. Both Intel and AMD are introducing such support in their processors, and patches to enable that support have been posted for consideration; the two manufacturers have taken somewhat different approaches to the problem, though. 

#### Intel's Software Guard Extensions

Intel's offering is called "Software Guard Extensions," or SGX; details can be found on [the SGX web page](<https://software.intel.com/en-us/sgx>). SGX is built around the idea of creating "enclaves" of protected code and data. One or more ranges of physical memory are set aside as the "enclave page cache"; the contents of that memory (whether data or code) are only accessible to code that is, itself, located within the enclave. That code is callable from outside the enclave, but only via a set of entry points defined when the enclave is set up. 

Memory within the enclave is encrypted using an engine built into the processor itself; the key that is used is generated at power-on and is not available to any running code. As a result, according to Intel's page, the contents of the enclave are ""protected even when the BIOS, VMM, OS, and drivers are compromised"". Those will certainly be appealing words for anybody who has despaired of ever preventing the compromise of any of those components. 

This mechanism seems to be aimed at protecting relatively small ranges of memory; its overhead is apparently too high to do more than that. So, for example, one might load a private key and the code to sign data with that key into an enclave. Thereafter, it will be possible for an application to use the key to create signatures, but nobody can gain access to the key itself, even if the kernel itself is compromised. 

The [SGX patch set](<https://lwn.net/Articles/685045/>), posted by Jarkko Sakkinen, creates a new device (`/dev/sgx`) which supports a number of `ioctl()` calls to control the feature. Interestingly, there are no capability checks on the `ioctl()` calls themselves; anybody who can open the device can set up enclaves — and the default permissions allow access to everybody. A new enclave is created with `SGX_IOCTL_ENCLAVE_CREATE`, and pages of data are added to it with `SGX_IOCTL_ENCLAVE_ADD_PAGE`. Things are then made ready to run with `SGX_IOCTL_ENCLAVE_INIT`. That last operation requires passing in an initialization token containing a hash of the enclave data and an appropriate signature. There does not appear to be a way to delete an enclave once it has been established. 

The "appropriate signature" part turns out to be a bit of a sticking point, in that said signature must come from Intel itself. In other words, it is not currently possible for the owner of a system to set up and use an enclave without getting Intel's approval and agreeing to [a set of conditions](<https://software.intel.com/en-us/articles/intel-sgx-product-licensing>). That means, as Ingo Molnar [put it](<https://lwn.net/Articles/686810/>), that ""it only allows the execution of externally blessed static binary blobs"". Needless to say, there is no shortage of opposition to that idea in the kernel community; Ingo went on to say: 

I don't think we can merge any of this upstream until it's clear that the hardware owner running open-source user-space can also freely define/start his own secure enclaves without having to sign the enclave with any external party. I.e. self-signed enclaves should be fundamentally supported as well. 

The discussion on the list [suggests](<https://lwn.net/Articles/686811/>) that Intel does plan to eventually make the feature work without the need for third-party signatures, but it is not clear when that might happen. Meanwhile, there is another roadblock, in that the patches do not actually work — one cannot actually run code in a protected enclave under Linux. Instead, enclaves can only run in the "debug mode," where it's possible to read and manipulate data inside the enclave from the rest of the system. That, obviously, detracts from the utility of the feature. It's not entirely clear why this limitation is in place. 

Jarkko had wanted to place the code into the staging tree, where developers could play with it until it was actually made to work, but there is no visible community interest in going along with that plan. So the SGX patches are going to have to wait for some time until (1) they actually work, and (2) the hardware can support enclaves signed by arbitrary parties. 

#### Secure Memory Encryption

AMD's technology is called "Secure Memory Encryption" (or SME); a description can be found in [this white paper [PDF]](<http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2013/12/AMD_Memory_Encryption_Whitepaper_v7-Public.pdf>). The [patch set](<https://lwn.net/Articles/685215/>) from Tom Lendacky adding support for SME came out, by some coincidence, one day after the SGX patches were posted. 

SME is, in a sense, a simpler mechanism. Rather than establishing enclaves, a system with SME simply marks a range of memory (even all of memory) for encryption by setting a bit in the relevant page-table entries. The memory controller will then encrypt all data stored to those pages using a key generated at power-on time; all data read from the range will be transparently decrypted. No code running on the processor (not even the kernel) has access to the encryption key. Enabling encryption is said to slightly increase memory latency, but the white paper suggests that the performance impact will normally be quite small. 

The SME approach, thus, will not protect memory from an attacker who has compromised the kernel; from the point of view of a running system, the memory might as well not be encrypted at all. Instead, it is intended to protect against cold-boot attacks, snooping on the memory bus, and the disclosure of transient data stored in persistent-memory arrays. 

SME can be used as the base for another feature, though, called "Secure Encrypted Virtualization" (SEV), where each virtual machine gets its own key. At that point, the value of the feature will, if it functions as advertised, be significantly higher; it can protect virtual machines from each other, keeping their contents secure even if one of them manages to compromise the host system. Indeed, it should be able to protect virtual machines from the hypervisor itself. In a world where everything seems to be moving toward virtual machines running on shared cloud infrastructure, this kind of protection would be a useful enhancement to the security of the cloud as a whole. 

The current patch set only implements SME, though, leaving SEV for the future. If the system is booted with the `mem_encrypt=on` command-line parameter, encryption will be enabled for all of physical memory, with a few exceptions. Video memory, for example, should not be encrypted; the device tree loaded at boot, if any, will also need to be accessed in the clear. Beyond that, encrypted memory will be used throughout, with most of the system being entirely unaware that the feature is in use. 

The SME patches do not have the signature-related issues that came up with SGX, but this feature was not universally acclaimed either. Andy Lutomirski detailed [several ways](<https://lwn.net/Articles/686812/>) that he thinks the feature could be broken before concluding: ""But I guess it's better than nothing."" Paolo Bonzini [added](<https://lwn.net/Articles/686813/>) that the SEV feature is ""very limited unless you paravirtualize everything"" and worried that it is being oversold as a general mechanism for the securing of virtual machines. He suggested that the work should maybe not be merged in its current form. 

Tom [acknowledged](<https://lwn.net/Articles/686814/>) that the technology has limitations in its current form, saying: 

In this first generation of SEV, we are targeting a threat model very similar to the one used by SMEP and SMAP. Specifically, SEV protects a guest from a benign but vulnerable hypervisor, where a malicious guest or unprivileged process exploits a system/hypervisor interface in an attempt to read or modify the guest's memory. But, like SMEP and SMAP, if an attacker has the ability to arbitrarily execute code in the kernel, he would be able to circumvent the control. AMD has a vision for this generation of SEV to be foundational to future generations that defend against stronger attacks. 

There have been no statements from the x86 maintainers regarding whether the SME patches would be merged, but it would not be surprising if their approach were similar to the one they have taken with the SGX patches: they will be seriously considered when the full functionality is present and that "vision" has been implemented. 

The end result of all this discussion is that we will probably not see support for either manufacturer's memory-encryption technology in a near-term kernel. But the direction that hardware development is taking offers some encouragement. Those of us who have despaired of ever truly securing our software may well be right; we need levels of defense that come into play when the software has failed. Done right, hardware-based defenses can come to the rescue here without taking away our power to secure and control our own systems. Once the hardware reaches that point, Linux will certainly be able to take advantage of those capabilities.
