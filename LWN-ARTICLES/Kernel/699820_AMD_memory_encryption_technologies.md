---
title: AMD memory encryption technologies
url: https://lwn.net/Articles/699820/
date: "September 8, 2016"
category: "Memory management-Memory encryption"
author: "By Jake Edge September 8, 2016 Linux Security Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
September 8, 2016

* * *

[Linux Security Summit](<https://lwn.net/Archives/ConferenceByYear/#2016-Linux_Security_Summit>)

Today's virtual machines (VMs) have a variety of protections from their brethren, but hypervisor or kernel bugs can allows guests to access the memory of other guests. In addition, providers of VMs can see the memory of any of the guests, so users of public clouds have to place a lot of trust in their provider. AMD has some upcoming features in its x86 processors that will encrypt memory in ways that alleviate some of these problems. David Kaplan gave a presentation about these new technologies at the Linux Security Summit in Toronto. 

[ ![\[David Kaplan\]](https://static.lwn.net/images/2016/lss-kaplan-sm.jpg) ](<https://lwn.net/Articles/699853/>)

The motivation for these features is the cloud environment. Currently, the hypervisor must enforce the isolation between guests through a variety of means: hardware virtualization support, page tables, VM intercepts, and so on. But sometimes those break down, leading to various vulnerabilities that allow guests to access other guests, which is scary, he said. 

But users are required to trust their cloud providers since they have full access to all guest memory to extract secrets or even to inject code into VMs. The cloud providers would rather not have that power, Kaplan said; they do not want to be able to see their customers' data. For one thing, that protects the providers from "rogue admin" attacks, where a disgruntled employee uses the unwanted access to attack a customer. 

That kind of attack, as well as those where a guest gets access to another guest's memory, are "user-access attacks", he said. AMD is targeting those as well as "physical-access attacks", where someone with access to the hardware can probe the physical DRAM interface or freeze and steal the memory chips (e.g. a [cold boot attack](<https://en.wikipedia.org/wiki/Cold_boot_attack>)). How important it is to resist those and other, similar attacks depends on who you talk to, he said. 

There are two separate features—Secure Memory Encryption (SME) and Secure Encrypted Virtualization (SEV)—that both use the same hardware support that will be provided in upcoming processors. That support includes an AES-128 hardware engine inline with the RAM and memory controller so that memory can be encrypted and decrypted on the way in and out of the processor with "minimal performance impact". The data inside the processor (e.g. registers, caches) will be in the clear; there will just be a "little extra latency" when RAM is involved. 

All of the keys will be managed within the SoC by the AMD Secure Processor, which is a separate 32-bit ARM Cortex A5 that is present on recent SoCs. It runs a secure (closed source) operating system and enables hardware-validated boot. It is used in some laptops as a firmware [Trusted Platform Module](<https://en.wikipedia.org/wiki/Trusted_Platform_Module>) (TPM). The secure processor will only run AMD-signed code; it also provides cryptographic key generation and management functions. 

Of the two features, SME is the simplest. It uses a single key that is generated at boot time using the random-number generator to transparently encrypt pages that have been marked with a special "encrypted" bit in the page-table entry. The operating system or hypervisor manages which pages will be encrypted in RAM by use of that bit. There is support for hardware devices to DMA to and from encrypted memory as well. SME is targeted at thwarting physical-access attacks, since the contents of memory will be inaccessible without the key that is not accessible outside of the secure processor. 

SEV, on the other hand, is more complicated. It has multiple encryption keys in the design and is meant to protect guests' memory from each other and from the hypervisor. The eventual goal, Kaplan said, is for the hypervisor to have no view into the guest. 

There are keys for the hypervisor and for each VM, though groups of VMs could share keys and some VMs might be unsecured. SEV cryptographically isolates the guests and hypervisor to the point where cache lines (which are unencrypted) are tagged with an ID that specifies which address space they belong to; the processor will prevent guests from accessing the cache of other guests. 

The owner of a guest is a "key player" in using SEV, Kaplan said. Information like secrets and policies will need to be transferred to the secure processor using the hypervisor to transport that data. Since the hypervisor is untrusted in this model (so that cloud providers do not have access to customer secrets), the guest owner will create a secure channel to the secure processor (through the hypervisor) using Diffie-Hellman (DH) key exchange. 

Launching a guest is a somewhat complicated process; Kaplan's [slides [PDF]](<http://events.linuxfoundation.org/sites/events/files/slides/AMD%20x86%20Memory%20Encryption%20Technology%20LSS%20Slides.pdf>) may be of interest for those who want more details. The hypervisor begins by loading an unencrypted BIOS or OS image into memory. The guest owner then supplies their DH key and the hypervisor facilitates the creation of a secure channel between the guest owner and the secure processor (without being able to eavesdrop on the traffic). Part of that exchange will provide the guest owner with a certificate that allows them to prove that they are truly talking to the secure processor. 

The hypervisor then allocates an "address space identifier" (ASID), which is what identifies the guest (and the key for that guest's memory). That ASID is provided to the secure processor with a request to generate or load a key into the AES engine and to encrypt the BIOS/OS image using that key. The hypervisor then sets up and runs the guest using the ASID assigned; the memory controller, AES engine, and secure processor will work together to ensure that the memory is encrypted and decrypted appropriately. 

The hypervisor will also send a "launch receipt" to the user that includes a measurement (hash) of the image and some platform authentication information. If the user is provided with the right measurement, they can then provide secrets like disk encryption keys to the guest over a secure channel (e.g. TLS). 

There are two levels of page tables: one for the guest and one for the hypervisor. The guest tables determine whether the memory is private or is shared with the hypervisor. All executable pages are private (no matter the setting), as are the guest's page tables. Data pages can be either, but DMA must use shared pages. 

A common question that is asked is in regards to the ASID: couldn't the hypervisor "spoof" a different ASID? The answer is that it could, but it wouldn't really gain it anything. If it tries walking the guest page tables or executing code using the wrong key, it will not be particularly successful. SEV is meant to block a range of attacks, both physical and user access; the intent is to reduce the attack surface even more in coming years. 

In order to use SEV, both hypervisors and guests will need to change to support it. There are a number of software components required, some that AMD expects to ship and others that it is working with the open-source community on. The secure processor firmware is distributed in binary form and the source is not public. There is a Linux driver to support the secure processor that has been posted for review. The open-source hypervisor support is also being worked on. 

There was a question about why AMD had not used the TPM API for its secure processor. Kaplan said there was interest in a simpler API that focused on the VM launch cycle. But the API is available and is only in beta at this point, so those interested should comment. Also, as is often the case with processor features, Kaplan was unable to say when SoCs with either feature would be available. 

[I would like to thank the Linux Foundation for travel support to attend the Linux Security Summit in Toronto.]
