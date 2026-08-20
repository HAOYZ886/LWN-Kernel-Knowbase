---
title: Protecting against TPM interposer attacks
url: https://lwn.net/Articles/1064685/
date: "April 6, 2026"
category: Security
author: "By Jake Edge April 6, 2026 SCALE"
---

By **Jake Edge**  
April 6, 2026

* * *

[SCALE](<https://lwn.net/Archives/ConferenceIndex/#Southern_California_Linux_Expo-2026>)

The [Trusted Platform Module](<https://en.wikipedia.org/wiki/Trusted_Platform_Module>) (TPM) is a widely misunderstood piece of hardware (or firmware) that lives in most x86-based computers. At [SCALE 23x](<https://www.socallinuxexpo.org/scale/23x>) in Pasadena, California, James Bottomley gave a presentation on the TPM and the work that he and others have done to enable the Linux kernel to work with it. In particular, he described the problems with interposer attacks, which target the communication between the TPM and the kernel, and what has been added to the kernel to thwart them. 

Bottomley introduced himself as a kernel developer and maintainer who worked on containers for around ten years before he joined Microsoft as an open-source evangelist. ""I enjoy studying how open-source systems work."" That was all just background, he said, because his talk was unrelated to any of that. 

The TPM has ""suddenly gotten a very bad rap"" due to the requirements of Windows 11, ""which obviously has nothing to do with me"", he said to chuckles around the room. He started looking into TPMs way back in 2011 after the [compromise of the kernel-development infrastructure](<https://lwn.net/Articles/461552/>) (i.e. kernel.org); part of the early efforts to better protect kernel.org used a hardware authentication device that could only store a single key, which was inconvenient since he had far more keys. He found out that the TPM [can be used as a key store](<https://blog.hansenpartnership.com/using-your-tpm-as-a-secure-key-store/>); since most laptops already had a TPM, his goal was simply to use the device to store his keys. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

So, 16 years later, he came to SCALE to talk about getting the TPM to work with the kernel, but he has also written code for various tools to allow them to use the TPM for their keys. That includes code in the [GNU Privacy Guard](<https://gnupg.org/>) (GPG) and a provider for [OpenSSL](<https://www.openssl.org/>) to allow them to work with the TPM. He connected the OpenSSL provider to [OpenSSH](<https://www.openssh.org/>), but the maintainers of that tool rejected the patches because they no longer use OpenSSL in favor of [LibreSSL](<https://www.libressl.org/>); that library has removed the pieces needed for his patch to work, however. 

#### TPM hardware

The TPM is often a discrete chip that sits on some hardware bus, such as the [Low Pin Count](<https://en.wikipedia.org/wiki/Low_Pin_Count>) (LPC) or [I2C](<https://en.wikipedia.org/wiki/I2C>) bus, in the system. It can also be a firmware TPM, as it is on his Dell XPS laptop, which runs on a separate CPU that is part of the management engine of the system. The firmware TPM communicates with the main CPU using a memory-based mailbox. The US National Security Agency (NSA) has mandated discrete TPMs in order to comply with its standards, so many laptop makers use a discrete chip, he said. 

[ ![\[James Bottomley\]](https://static.lwn.net/images/2026/scale-bottomley-sm.png) ](<https://lwn.net/Articles/1065940/>)

A discrete TPM is often [susceptible to simple probing of the bus](<https://docs.kernel.org/security/tpm/tpm-security.html>) it sits on, in what is known as an interposer attack. With a small amount of hardware, all of the traffic on the bus can be captured. Since it is an internal bus for the system, and encryption has a cost that seemed like it would be wasted, the traffic is in clear text. 

The disclosure of the attack led TPM users to shift their thinking: since the channel to and from the TPM was potentially compromised, sending secrets in clear text did not make sense any longer. The only attack demonstrated so far is a passive interposer, he said, which is easily defeated by encrypting TPM communication. An active interposer could potentially sit in the middle and alter the communication to and from the real TPM; that would allow it to participate in the encryption so it could see the real data. Handling that kind of attack is rather harder. 

In order to do encryption, however, a TPM session must be established, which sets up a shared key between the TPM and the user. The encryption mechanism is a complex one, where the session key is ""basically a salt where key-derivation functions are applied to nonces that are exchanged on the bus"". It is a ""really really complicated scheme, but the bottom line is, if you know the session key, you can decrypt the traffic"". 

TPM sessions were added for all Linux kernel TPM transactions starting with Linux 6.10. That was around two years ago, Bottomley said. He wrote all of that code, which is now a source of complaints because of the overhead introduced by the encryption. TPMs are typically low-powered devices and they are slow to do encryption and decryption, but encryption does protect against a passive interposer. 

There is a public key for the TPM that is used in the process of creating the session key; if active interposers are considered, though, there is a problem in obtaining that public key. Most toolkits simply ask the TPM for the key: ""How do I know that the public key it gave me is the one I'm supposed to have?"" 

The TPM-session work that has gone into the kernel is only used when the kernel itself needs to talk to the TPM, so it is the user in that case. If an application, systemd, say, wants to establish an encrypted session, it must do the session startup for itself as the user. In practice, user-space applications use a library known as the [trusted software stack](<https://trustedcomputinggroup.org/resource/creating-complete-trusted-computing-ecosystem-overview-trusted-software-stack-tss-2-0/>) (TSS) to talk to the TPM. 

There are two main TSS implementations, one by Intel and another by IBM; both simply ask the TPM for its public key. As noted, that would allow an active interposer to present a public key it controls and pass the traffic through to the real TPM, while siphoning off secrets that are being "protected" by the TPM. Application programmers who need to communicate with the TPM are drawn to using a TSS to avoid the ""astronomically"" difficult TPM programming, but those libraries do not handle this problem correctly—or even alert users that there is a potential problem. There is no verification done to ensure that the public key provided is the right key to use. 

There are other interposer problems, as well. ""Even a passive interposer just sitting on the LPC bus can actually toggle the TPM's reset line."" That can cause problems because one of the jobs of the TPM is to maintain the [platform configuration registers](<https://en.wikipedia.org/wiki/Trusted_Platform_Module#Platform_integrity>) (PCRs) that are used to prove that the code that has been run is what is expected, which is important for integrity-checking purposes. The PCRs are meant to be extended with hash values of the code executed, but toggling the reset line sets all of the PCRs back to zero. An attacker who can reset them can then extend them anew by feeding in all of the proper hashes, even though the code in question was not executed, causing an undetectable integrity breach on the system. 

#### Trusting the public key

Developers need to be able to establish trust in a TPM public key, but the established mechanism ""is so complicated that no one ever does it"", including the TSS libraries. All discrete TPMs ship with a [certificate from the manufacturer that validates the Endorsement key](<https://trustedcomputinggroup.org/wp-content/uploads/IWG-EK-CMC-enrollment-for-TPM-v1-2-FAQ-rev-April-3-2013.pdf>) (EK), which is resident in the TPM. That [X.509](<https://en.wikipedia.org/wiki/X.509>) certificate can be validated because it is signed by manufacturer's master certificate authority (CA), so an EK cannot be forged, which provides a TPM key that can be trusted. 

Unfortunately, the EK cannot be used to validate any of the other keys in the TPM. Primary keys are generated by the TPM using a [key derivation function](<https://en.wikipedia.org/wiki/Key_derivation_function>), which allows it to generate different kinds of keys from its internal seed, but those keys are either encryption or signing keys. The device can only certify objects (e.g. keys) using signing keys, but the EK can only certify _encryption_ keys, so it cannot certify signing keys. 

That raises a question: why is there no signing key that can be validated in the way that the EK can be? ""Turns out, it's all the FSF's [[Free Software Foundation's](<https://www.fsf.org/>)] fault"", Bottomley said. The [Trusted Computing Group](<https://trustedcomputinggroup.org/>) (TCG), which is the organization behind the TPM standards, ""got badly burned by the FSF backlash against something they called 'Treacherous Computing' way back in the early 2000s"". The concern was that the TPM would be used to effectively lock users out of being able to do what they want with their computers (e.g. preventing video decryption); ""it's all actually untrue, the TPM is too weak [of] an encryption engine to do this, but it was seen as 'the spy in your computer'"" by the FSF and others. 

The operating system is what governs access to the TPM, he said, ""if you own the operating system on your computer, you're the one guarding access to the TPM"". There were plans for non-open-source operating systems ""to use TPMs for various nefarious purposes"", he said, but those plans were rapidly killed by the Treacherous Computing campaign. 

To counter the somewhat-misplaced criticism of the TPM as an agent of others in PCs—TPMs cannot communicate at all without operating-system help—the TCG added some privacy-preserving features to the device. One of those was to restrict manufacturers from shipping signing certificates. Because a signing certificate would uniquely identify a particular TPM, thus become a way to track people online via their TPM operations. 

Instead, the TCG allows the TPM to create an arbitrary number of signing keys, called Attestation keys (AKs). Each AK can be certified as coming from a particular EK (thus TPM) by way of a new organization called a PrivacyCA. It would be a third-party CA that is trusted by the operating system, user, and whatever entities need to validate objects from the TPM; it would be the only party that is able to make the connections for the tracking, which is why it needs to be trusted. 

""The PrivacyCA idea has been around for ten years; none exist today."" So it is a system for validating keys on the TPM that is supported by all TPMs but is unusable in practice to thwart the interposer attacks. For his purposes, though, the PrivacyCA would not help; the kernel needs to validate the keys at early boot time and there is no internet available then. 

Another possible angle would be for the kernel to use the EK as its encryption key for communicating with the TPM, but it requires the internet to verify the certificate with the manufacturer's CA. If he could collect up all of the root certificates for all of those CAs, they could be used for the verification since the kernel does have code that can validate a chain of X.509 certificates. At the time he was doing the work, there were many possible TPM manufacturers; these days, there are only around three or four, making collecting the root certificates less daunting. But there is still another problem: firmware TPMs, like the one in his laptop, do not have EK certificates at all. 

#### TPM trivia

So, solving the problem for the kernel required a different approach. Describing it required a quick lesson in what Bottomley called ""more TPM trivia"". 

The [TPM 2.0](<https://en.wikipedia.org/wiki/Trusted_Platform_Module#TPM_2.0>) standard has four hierarchies that are aimed at different users of the chip. They are: the platform hierarchy, which is only accessible to the BIOS, the endorsement hierarchy, where the EK is stored but not used otherwise, the owner or storage hierarchy, where TPM-user keys are normally stored, and the NULL hierarchy, which is not normally used. There is an interesting property of the NULL hierarchy, he said, in that its seed is reinitialized every time the TPM is reset. Keys created in that hierarchy are only usable until the next reset, which is fine for short-lived operations like encrypting the TPM traffic during boot. In addition, if an interposer causes a TPM reset during boot, the kernel will detect it because the kernel's encryption to and from the TPM will fail. 

The underlying problem of ensuring that the key is legitimate has not gone away, however. There is no straightforward way to ensure that the NULL key is actually from the TPM and not from an interposer. What he has come up with uses the "name" of the NULL key, which is simply a hash of the public portion of the key, so it is a unique ID for the key. That name gets exported in a sysfs file, which can be verified using a more sophisticated mechanism after the kernel boots. 

His [openssl_tpm2_engine project](<https://git.kernel.org/pub/scm/linux/kernel/git/jejb/openssl_tpm2_engine.git/tree/README>) has a tool, [`attest_tpm2_primary`](<https://manpages.opensuse.org/Tumbleweed/openssl_tpm2_engine/attest_tpm2_primary.1.en.html>), that can be used to do that verification. It can create an EK signing key that can then be verified as coming from the EK that corresponds to the certificate that is signed by the manufacturer. The name of that signing key could be stored somewhere read-only as part of the operating-system installation process, because that verification only needs to be done once. Since the owner of the system is the one creating the EK signing key, and it does not matter if they can track themselves, the privacy concerns that the TCG had, thus the need for a PrivacyCA, are not present, Bottomley said. 

#### Threat models

In order for the tool to be trustworthy, the operating system has to be trusted. ""The TPM people will tell you that the TPM is the only way of trusting your operating system, but that's not true"" as secure boot can provide that validation; he noted that developers from the TPM and secure-boot communities often do not work together well. With [unified kernel images](<https://uapi-group.org/specifications/specs/unified_kernel_image/>) (UKIs), secure boot can validate the initrd, as well. Full-disk encryption completes the picture, because the decryption password validates the contents of the filesystem. 

So an attack on an unattended laptop (a so-called "evil maid" attack) that installs an interposer can be detected unless the interposer device, ""which is a fairly weak thing"", can also compromise the kernel enough to impact the TPM validation, he said. The operating system must be trusted in order to validate the TPM. ""If somebody steals your laptop and turns off secure boot, substitutes their own kernel that defeats all of this, and manages to get your disk encryption key, it's game over."" In addition, unlocking the TPM and its keys based solely on the measured state in the PCRs is dangerous; a second factor (e.g. a PIN) should be required to ensure that the system has not been compromised by operating-system replacement. 

Bottomley then demonstrated a virtual machine (VM) that was talking to the [IBM software TPM](<https://ibmswtpm.sourceforge.net/ibmswtpm2.html>), which is useful for testing, he said. He also started up an [active interposer written in Python](<https://git.kernel.org/pub/scm/linux/kernel/git/jejb/tpm2-interposer.git/tree/tpm2-interposer.py>) that sat between the VM and the TPM. The VM was set up to unlock the TPM keys based on the PCR values; he was able to log into the system and see the value for the NULL hierarchy key name. The software TPM has a mechanism to generate a EK certificate, so he could create the EK signing key and its name would have been stored in a file on the encrypted disk at install time. He was able to use `attest_tpm2_primary` to show that the EK signing key could be validated with the EK certificate and that the NULL hierarchy key name was correct, which validated the boot. 

During all of that, the interposer was simply passing the data back and forth, without being able to inspect or modify it because the kernel and TPM were encrypting it. He then changed the disk encryption password using systemd, which uses the storage hierarchy key that it cannot validate using the kernel's mechanism. That allows the interposer to use its own key so that it can see all of the traffic between systemd and the TPM. That means it could display (or modify) the new disk password systemd was using. 

There was more to the demo, which was rather complicated as one might expect. The [YouTube livestream](<https://www.youtube.com/watch?v=dja8X8mFaNE>) for the talk starts a few minutes late, but the demo comes nearly 30 minutes in, so interested readers can have a look. 

He concluded the talk by noting that the NULL hierarchy provides a means to fully protect the kernel from TPM-reset attacks, and the after-boot verification can prove that the keys from the TPM can be trusted—as long as the kernel and user space are trusted (via secure boot, for example). Any TPM-using tool could use the NULL mechanism and compare the key that the kernel exports to ensure that it is talking to the real TPM, which would provide protection against interposers and reset attacks. Doing that work in each application would be burdensome, Bottomley told me later, so it would make sense to do it in the TSS libraries, though it is not clear if that is on the horizon. 

It was an interesting look into the rather fiddly world of the TPM and how it can be used to secure our systems. One guesses that the interposer attack is fairly rare, except possibly against seriously high-value targets, but it is important to be able to defend against it. The kernel can do so, at least on systems with secure boot and UKIs, but user-space tools will want to catch up and Bottomley has shown a possible path to get there. 

[Thanks to LWN's travel sponsor, the Linux Foundation, for its travel funding to attend SCALE in Pasadena.]
