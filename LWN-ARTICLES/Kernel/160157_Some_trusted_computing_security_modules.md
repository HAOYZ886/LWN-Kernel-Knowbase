---
title: Some trusted computing security modules
url: https://lwn.net/Articles/160157/
date: "November 15, 2005"
category: "Integrity measurement architecture; Security-Security modules; Trusted computing"
author: 
---

IBM has been pushing the use of the "trusted platform module" chip found in its laptops (and on other systems as well) for some time; see [this report from OLS 2005](<http://lwn.net/Articles/144681/>) for a summary of the benefits they see from trusted computing. Now IBM's developers have posted [a new set of security modules](<http://lwn.net/Articles/160126/>) which make use of the TPM to lock down a system. The three modules are: 

  * The **simple Linux integrity model** , or SLIM. This module associates two attributes with every process and every file: the integrity level and the privacy level. The integrity levels are "system," "user," and "untrusted." Any process may read or execute any file with an equal or higher integrity level (subject to the usual permissions). Read and execute access to lower-integrity files is also allowed, but, as a result, the process will, itself, be demoted to the lower level. Writing files with a higher integrity level is not allowed. The integrity levels thus implement a form of simple, automatic sandboxing; if a process touches untrusted resources, it also loses trust and has a lowered ability to change things elsewhere on the system. Network sockets, incidentally, are always considered to have an "untrusted" integrity level. 

The privacy level has four levels: public, user, user-sensitive, and system-sensitive. Processes can read files of equal or lower sensitivity. If, instead, a process reads a higher-sensitivity file, its own sensitivity level is raised to match. Writing lower-sensitivity files is not allowed. This "high watermark" mechanism is intended to prevent the leakage of secret data to less-trusted contexts. 

  * The SLIM module, like SELinux, depends on the extended attributes of a file to make security decisions. But what if something is able to change those attributes? The **extended verification module** (EVM) is an attempt to keep that from happening. EVM creates its own extended attribute on each file which is an [HMAC hash](<http://en.wikipedia.org/wiki/HMAC>) of the file's contents and attributes. If the file and the HMAC fail to match, EVM will deny access to the file. 

One might argue that EVM's hash is no less susceptible to tampering than the other attributes on the file. The difference is that EVM uses the hardware TPM module to sign the HMAC result. The TPM will only perform this operation if it is satisfied that the proper "secure boot" rituals have been followed, and that the integrity of the running system has not been compromised. Since the TPM key is specific to that particular chip, it is not possible to remove the drive and forge HMACs on a different system. If the trusted boot chain, starting with the BIOS, holds, there should be a high level of assurance that the system's files and their attributes have not been tampered with. 

  * The third module is the **integrity measurement architecture**. LWN readers have [seen IMA before](<http://lwn.net/Articles/137306/>), so that discussion will not be repeated. In short, IMA is a remote attestation feature which can provide a convincing proof that a system is running (only) well-known, trusted versions of approved software. 

The IMA module was not well received when it was last posted. The developers hope that the largest objections have been addressed, and that the set of TPM-related modules as a whole can be considered, eventually, for merging. Before reaching that point, however, these modules have another obstacle to overcome: they rely on the ability to run multiple Linux security modules in a "stacked" mode. Stacked security modules have been [a contentious issue](<http://lwn.net/Articles/110432/>) for some time, and that capability has never been merged. The developers claim that the new modules will make the case for stacking, but that conversation has yet to take place.
