---
title: "OLS: Linux and trusted computing"
url: https://lwn.net/Articles/144681/
date: "July 22, 2005"
category: Integrity measurement architecture; Security; Trusted computing
author: 
---

The term "trusted computing" tends to elicit a suspicious response in the free software community. It has come to be associated with digital restrictions management schemes, locked-down systems, and similar, untrustworthy mechanisms. At the 2005 Ottawa Linux Symposium, Emily Ratliff and Tom Lendacky discussed the state of trusted computing support for Linux and tried to show how this technology can be a good thing to have. Trusted computing does not have to be evil. 

At the lowest level, trusted computing is implemented by a small chip called the "trusted platform module" or TPM. The Linux kernel has had driver support for TPM chips since 2.6.12; a couple of chips are supported now, with drivers for others in the works. Many systems - laptops in particular - are currently equipped with TPM chips, so this is a technology which Linux users can play with today. 

A TPM provides a number of features to the host system. It includes a protected memory area, and a restricted set of commands which can operate on that area. "Platform configuration registers" (PCRs) are a special sort of hashed accumulator which can be used to track the current hardware and software configuration of the system. The TPM also includes a cryptographic processor with a number of basic functions: a random number generator, SHA hash calculator, etc. And there is some non-volatile RAM for holding keys and such. 

A TPM-equipped system requires support in the BIOS. Before the system boots, the BIOS will "measure" the current hardware state, storing the result in a PCR. The boot loader will also be checksummed, with the result going into another PCR. The boot loader is then run; its job is to stash a checksum of the kernel into yet another register before actually booting that kernel. Once the kernel is up, the "trusted software stack" takes charge of talking to the TPM, providing access to its services and keeping an eye on the state of the system. Systems which provide a TPM typically also include the needed BIOS support; this support could also be added by projects like FreeBIOS and OpenBIOS. There are [versions of the Grub bootloader](<http://www.prosec.rub.de/trusted_grub.html>) which can handle the next step; LILO patches also exist. Once the kernel is booted, the TPM driver takes over, with the user-space being handled by the [TrouSerS](<http://trousers.sourceforge.net/>) TSS stack. 

TrouSerS makes a number of TPM capabilities available to the system. If the TPM has RSA capabilities, TrouSerS can perform RSA key pair generation, along with encryption and decryption. There is support for remote attestation functionality (more about that momentarily). The TSS can be used to "seal" data; such data will be encrypted in such a way that it can only be decrypted if certain PCRs contain the same values. This capability can also be used to bind data to a specific system; move an encrypted file to another host, and that host's TPM will simply lack the keys it needs to decrypt that file. Needless to say, if you make use of these features, you need to give some real thought to recovery plans; there are various sorts of key escrow schemes and such which can be used to get your data back should your motherboard (with its TPM chip) go up in flames. 

The TrouSerS package also provides a set of tools for TPM configuration tasks. However, a number of BIOS implementations will lock down the TPM before invoking the boot loader, so TPM configuration is often best done by working directly with the BIOS. There is also a PCKS#11 library; PCKS#11 is a standard API for working with cryptographic hardware. 

At the next level is the integrity measurement architecture (IMA) code. IMA was [covered on the LWN Kernel Page](<http://lwn.net/Articles/137306/>) last May; look there for the details. In short: IMA uses a PCR to accumulate checksums of every application and library run on the system since boot; this checksum, when signed by the TPM, can be provided to another system to prove that the measured system is running a specific list of software, that the programs have not been modified, and that nothing which is not on the list has been run. If the chain of trust (starting with the BIOS) holds together, a remote system can have a high degree of confidence that the list is accurate and complete. 

Since last May, the IMA code has been significantly reworked (it took a fair amount of criticism on the kernel list). Among other things, it no longer hooks in as a Linux security module. The next step, however, will be a security module; it is called the "extended verification module." It includes a fair amount of security enforcement policy. This module can, for example, check that the extended attributes on files have not been changed by any third party. SELinux makes heavy use of extended attributes; with this mechanism in place, an SELinux system can remain secure even if somebody moves the disk to a different system and makes changes to the SELinux labels. Once back on the original system, those changes will be detected. 

So why would a Linux user care about all of this? Some of the things that can be done with the TPM include: 

  * Key protection. A user can store GPG keys (or others) in the TPM and not have to worry about those keys being extracted and disclosed by a compromised application. 

  * System integrity checking. The measurement capabilities can be used to ensure that the binaries on the system have not been tampered with; it is a sort of Tripwire with hardware support. 

  * In the corporate environment, the remote attestation features provided by IMA can be used to keep compromised systems from affecting the company network. Simply require systems to provide their "measurement" before giving them access to the network, and any system which has, say, been infected with malware at a conference will be detected and locked out. 

  * Similarly, a conference attendee using an "email garden" terminal to access a mail server could, in the future, require that terminal to verify itself to the server before any sort of access is allowed. 

  * Attestation could be used in electronic voting machines to verify that they are running the proper (hopefully open source) software. 

And so on. The point is that there are legitimate uses for a hardware-based mechanism which can, with a reasonable level of confidence, verify that a system's software has not been compromised. 

On the other hand, this same technology has a number of other potential uses. It could be used by company IT cops to ensure that employees are not running "unapproved" software, be it games, unlicensed copies of proprietary software, or Linux. Remote attestation is a boon for companies like TiVo, which can use it to ensure that the remote system is running current software and has not been cracked. Providers of web services could be sure that you really _are_ running Internet Explorer. It does not take much imagination to come up with several unpleasant scenarios involving trusted computing and locked-down systems. 

What it comes down to is that "trusted computing," like computing itself, is a tool which can be used in many ways. One does not have to look very far to find people using Linux in ways that one, personally, might not approve of. The TPM hackers feel that, given that the technology is available, let's use it. Properly used, this hardware can help to ensure that we remain in charge of our systems, and that much, certainly, is a good thing.
