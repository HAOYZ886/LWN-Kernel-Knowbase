---
title: Toward measured boot out of the box
url: https://lwn.net/Articles/699551/
date: "September 8, 2016"
category: "Bootstrap process; Security-Integrity verification"
author: "By Jake Edge September 8, 2016 Linux Security Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
September 8, 2016

* * *

[Linux Security Summit](<https://lwn.net/Archives/ConferenceByYear/#2016-Linux_Security_Summit>)

Matthew Garrett began his [Linux Security Summit](<http://events.linuxfoundation.org/events/linux-security-summit>) talk by noting that the "security of the boot chain is vital" to having secure systems. It does not matter if the kernel can protect itself, referring to the [talk](<https://lwn.net/Articles/698827/>) just prior to his; if the boot process can be manipulated, those protections are immaterial. So, he wanted to present where things stand with regard to securing the boot chain. 

In the Linux world, UEFI Secure Boot is the primary boot protection mechanism; it requires that the bootloader be signed by a key that is trusted by the firmware or the system won't boot. There are also various solutions for embedded devices that are typically implemented by the system on chip (SoC). The trust is rooted in the firmware in either case; if someone can modify the firmware, all bets are off. 

[ ![\[Matthew Garrett\]](https://static.lwn.net/images/2016/lss-garrett-sm.jpg) ](<https://lwn.net/Articles/699851/>)

Beyond that, most of the existing mechanisms provide no way to prove that the verification of the code to be booted has been done. The running kernel has no way to know that it is running on a base that has been integrity checked—or even whether the kernel itself has been tampered with—any query it could make could be answered with a fake "yes". 

That kind of attack generally requires privileged access to the hardware, which is a hazard in its own right, so why would those kinds of attacks matter, he asked. One problem area is that there are providers of "bare metal" servers for users who want the convenience of the cloud without its usual performance penalty. Users of those systems will have root privileges, which will allow them to access the hardware, including potentially permanently changing the firmware to something malicious. 

He posited a scenario where an attacker would take out a large number of short-term leases on hardware at a site that is known to be used by the victim. Each system is then infected with malicious firmware and "returned" to the pool at the hosting company. Some of those systems will eventually be picked up by the victim; "Secure Boot will not help you" in that situation, he said. 

Another worrisome possibility is for laptops that are surrendered when passing through borders. Perhaps it is overly paranoid to be worried about permanent firmware changes being made at the border, he said, but it is at least worth thinking about. While there is not much that can be done to protect against hardware-based attacks (e.g. adding some malicious hardware to a laptop or server), most of the other kinds of attacks can be handled. 

#### TPM to the rescue

The [Trusted Platform Module](<https://en.wikipedia.org/wiki/Trusted_Platform_Module>) (TPM) is a bit of hardware that can help. When it was first introduced it got a bad reputation because it was "easy to portray it as a DRM mechanism", though it is difficult to deploy that way and no one has actually done so. TPMs are small chips, made by several different manufacturers, that are generally differentiated by their performance and amount of NVRAM storage they provide. TPM implementations also have "a bewildering array of different bugs", Garrett said. 

TPMs have several functions, but the one of interest for ensuring that the boot process has not been tampered with uses the platform configuration registers (PCRs). They are normally 16-24 registers that are not directly accessible outside of the chip; all access is mediated by the rules of the TPM. PCRs are 20 bytes long in TPM 1.2, which is the length of an SHA-1 hash; TPM 2.0 allows for multiple hash algorithms, so the number and size of the PCRs changes to support them. 

Ensuring tamper-free boot means that each step of the process must be "measured", which effectively means calculating a cryptographic hash of the binary. Each step in the boot process would measure the next, so the firmware measures the bootloader, the bootloader measures the kernel and initial ramdisk (initrd), and so on. The PCRs provide a tamper-proof mechanism to assist in the measurement process. 

One cannot store a value directly into a PCR; instead the TPM must be asked to store the value, which it does in a way that provides integrity to the result. Instead of just storing the value, which would allow any program with access to the hardware to set it to the "right" value, it concatenates the existing value in the PCR and the written value (typically the hash of the measured data) and hashes the result. So, in order to reproduce the value in a given PCR, the same measurements must be written to the register in the same order. 

There is also a log associated with the TPM. Each measurement adds an entry to the log that records what was measured and what the hash was. While untrusted code can overwrite the log, he said, that turns out not to be as much of a problem as it sounds. 

All x86 firmware has measurement capabilities, though sometimes there are problems with what they can measure. For example, there was firmware he encountered that would measure code that came from disk, but not code that came via a network boot, which kind of misses the point. But that firmware has since been fixed. 

#### Bootloader support

There is no Linux bootloader that supports measurement, however. At one time, [TrustedGRUB](<https://sourceforge.net/projects/trustedgrub/>) could be used, but it is now "old and busted"; it worked, but it "wasn't particularly nice", Garrett said. Rohde & Schwarz Cybersecurity have developed [TrustedGRUB2](<https://github.com/Rohde-Schwarz-Cybersecurity/TrustedGRUB2>), which supports using the TPM, but it has some shortcomings. In particular, it does not support UEFI or TPM 2.0. So, Garrett and others have added code to GRUB 2 to support measuring the kernel and other components at boot time (in [this GitHub repository](<https://github.com/coreos/grub>)). 

There is more to measure than just the kernel, however. The booted state of the system is affected by many other components and configuration files. The kernel command line is relevant, as is the GRUB configuration, since GRUB has a scripting interface that can make hardware changes. 

But putting each individual configuration piece into its own PCR does not scale because there are a limited number of them. So there is a need to reuse PCRs, but the final value of the PCR will depend on the order in which those items were measured. Trying to establish a strict ordering is something he would like to avoid. There is also the problem that unimportant changes to configuration files (e.g. comments) will still cause the final hash value to be different. For those and other reasons, using the PCRs that way is suboptimal, he said. 

Instead, though, the log file can be used. It can be overwritten with invalid data, but that can be detected by replaying the log and calculating the hashes independently. There are two formatting possibilities for the log messages that Garrett described. The first would log a description of the binary and its hash, which is fine for a small number of binaries. That doesn't work so well for configuration information, though, because it may have unimportant changes that alter the hash. For those, the log entry would contain the text that has been hashed in conjunction with its hash. 

Then there needs to be a policy file that describes the acceptable hashes for binaries as well as the allowable text for configuration (using regular expressions for parameters and the like). Creating that policy may be rather troublesome, though. His employer, CoreOS, builds the policy automatically for each release. The policy is not complete, however, since it needs known-good hashes for the firmware on the system and no firmware vendor he knows provides that information. So CoreOS users must extract the values from a known-good system, which will work fine unless the firmware is upgraded at some point. 

While it is easy for CoreOS to provide an initial RAM filesystem (initramfs) and its hash, other distributions build the initramfs on the user's system when the kernel or other components are updated. Timestamps then get into the binary, which means the hash is different for each. Some kind of generic initramfs using reproducible build mechanisms would alleviate that problem. 

There is also a question of where the boot data gets stored. If it is stored in the initramfs, that will change the hash, so he suggested using UEFI variables for some information and the TPM for keys. In a process known as "sealing", the TPM can store encrypted information that it will only decrypt if certain PCRs have the right values to show that the boot process has not been tampered with. Having sealed keys for [tpmtotp](<https://github.com/mjg59/tpmtotp>) (Time-based one-time password, TOTP, attestation using the TPM), disk encryption, or SSH would ensure that the data is only available to properly booted systems. 

One problem that has not yet been solved is handling firmware or operating system upgrades. There needs to be a mechanism to unseal values and reseal them based on the upgraded system. So far, no solution to that problem has been found. 

Intel's [Trusted Execution Technology](<https://en.wikipedia.org/wiki/Trusted_Execution_Technology>) (TXT) is supposed to make this all easier, he said, but that isn't the case. TXT is based on a dynamic root of trust, rather than the static root of trust used by TPM, which in theory would sidestep some of the problems that the TPM-based boot integrity has encountered. But TXT has "no meaningful support for Secure Boot" and it is also incompatible with runtime UEFI. In effect, Garrett said, TXT is not compatible with the way we boot operating systems. 

#### To do

There are still things that need to be done before this gets into the hands of users. Support for it needs to ship in bootloaders; the firmware in desktop systems is likely to have lots of different bugs that may cause systems using this feature not to boot, so there is a lot of testing work to be done there. Firmware vendors and distributions will need to start shipping known-good measurements. The firmware upgrade process will need to be integrated with updating the measurement information and there will need to be ways to create initramfs images deterministically. But we are getting closer to having measured boot right out of the box. 

One audience member wondered about the patches to GRUB 2 and whether those would be making their way upstream. Garrett said that he plans to do that; he has talked to Richard Stallman and convinced him that what was being done was "not intrinsically evil", which was met with audience applause. Garrett joked that he hoped that would find its way into his annual performance review. 

GRUB 2 has a new maintainer who is more active, he said, which should help getting this work upstream. There is one problem, however, in that the GRUB 2 project requires copyright assignment and some of the code comes from TrustedGRUB, which he can't assign. He is looking to resolve that since he does not want out-of-tree patches. 

[I would like to thank the Linux Foundation for travel support to attend the Linux Security Summit in Toronto.]
