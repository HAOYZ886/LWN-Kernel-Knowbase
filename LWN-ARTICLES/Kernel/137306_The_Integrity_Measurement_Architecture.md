---
title: The Integrity Measurement Architecture
url: https://lwn.net/Articles/137306/
date: "May 24, 2005"
category: "Integrity measurement architecture; Security-Integrity verification"
author: 
---

One of the many new features in the 2.6.11 kernel was a driver for "trusted platform module" (TPM) chips. This driver made the low-level capabilities of TPM chips available, but gave no indication of what sort of applications were envisioned for those capabilities. Reiner Sailer of IBM has now taken the next step with [a set of patches](<https://lwn.net/Articles/136834/>) implementing the "Integrity Measurement Architecture" (IMA) for Linux using TPM. 

IMA is a remote attestation mechanism, designed to be able to convince a remote party that a system is running (nothing but) a set of known and approved executables. It is set up as a security module, and works by hooking into the `mmap()` operation. Whenever a file is mapped in an executable mode (which is what happens when a program is run or a sharable library is mapped), the IMA hook will first perform and save an SHA1 hash of the file. On request, the IMA module can produce a list of all programs run and their corresponding hash values. This list can be examined by a (possibly remote) program to ensure that no unknown or known-vulnerable applications have been run. 

If a hostile application has managed to take over the system, however, it will be in a position to corrupt the list from the IMA module, rendering that list useless. This is where the TPM chip comes in. The TPM contains a set of "platform configuration registers" (PCRs) which are accessible to the the rest of the system only in very specific ways. The PCRs can be reset to zero only when the system hardware itself is reset. The host system can pass data to the TPM which is to be included in a given PCR; the TPM then computes a hash with the new information and stores the value in the PCR. A given set of values, if sent to a PCR in any order, will, at the end, yield the same final hash value. The TPM can provide that value on request; it can also be made to sign the hash value using a top-secret key hidden deeply within its tamper-proof packaging. 

The IMA module works by sending each hash it computes to a PCR on the TPM chip. When it provides the list of executables and hash values, it can also obtain and hand over a signed hash from the TPM. A remote party can then recompute the hash, compare it to what the TPM produced, and verify that the provided list is accurate. It is still possible for an intruder to corrupt the list, but it will then fail to match the hash from the TPM. It thus should be possible to remotely detect a compromised system. 

Of course, if an attacker can gain control of the kernel at boot time, before the IMA module has been initialized, the entire battle has been lost. The TPM designers have thought of this possibility, however; it is possible to set up hardware so that it will not boot a system in the first place unless the TPM approves of the code to be booted. 

There are numerous possible applications of this sort of capability. In a highly secured network, systems could refuse to talk to each other until each proves that it is running only approved software. Financial web sites could, if given access to this information, refuse access from systems running browsers with known security problems. The less flexible sort of Linux support provider could refuse to work on systems which have run programs which are not on The List Of Supported Applications. Corporate IT departments could get verifiable lists of which programs have run on each system. DRM-enabled software could refuse to unlock its valuable intellectual property if the system looks suspicious. And so on. 

In the short term, however, this code looks like it will need some work before it will be considered seriously for inclusion. James Morris has [questioned](<https://lwn.net/Articles/137310/>) the security module implementation, arguing that this functionality should be implemented directly in the kernel. Loading the IMA module also makes it impossible to use any other security module (such as SELinux), which may not enhance the overall security of the system. And Greg Kroah-Hartman [was unimpressed](<https://lwn.net/Articles/137311/>) with the quality of the code in general: 

Wow, for such a small file, every single function was incorrect. And you abused sysfs in a new and interesting way that I didn't think was even possible. I think this is two new records you have set here, congratulations. 

The IMA authors have now gone off to rework things. At some point, however, it seems likely that this sort of functionality will be available in Linux. Whether it will then be used to increase or restrict the freedom of Linux users remains to be seen. 

(For more information, see [the IBM tcgLinux](<http://www.research.ibm.com/secure_systems_department/projects/tcglinux/>) and [Trusted Computing Group](<https://www.trustedcomputinggroup.org/home>) pages).
