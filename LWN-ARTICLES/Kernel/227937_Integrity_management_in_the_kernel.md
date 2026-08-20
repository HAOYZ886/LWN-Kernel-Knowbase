---
title: Integrity management in the kernel
url: https://lwn.net/Articles/227937/
date: "March 28, 2007"
category: "Integrity measurement architecture; Security-Integrity verification"
author: 
---

Certain patches seem to pop up occasionally on the kernel lists for years. One of those is the whole integrity management patch set from IBM; these patches were last covered here in [November, 2005](<http://lwn.net/Articles/160157/>). They are back for consideration yet again. Integrity management still looks like it is not ready for inclusion into the mainline, but it is getting closer; at some point it will force consideration of some interesting questions. 

The core idea behind integrity management is providing some sort of assurance that the files on the system have not been messed with. David Safford [described it](<https://lwn.net/Articles/227939/>) this way: 

[B]asically this integrity provider is designed to complement mandatory access control systems like selinux and slim. Such systems can protect a running system against on-line attacks, but do not protect against off-line attacks (booting Knoppix and changing executables or their selinux labels), or against attacks which find weaknesses in the kernel or the LSM module itself. 

The current patches work, at the lowest level, by defining a new set of security module hooks for an "integrity provider." The provider can hook into system calls which access or execute files and check the integrity of those files; should it conclude that Bad Things have happened, access to the files can be denied. On top of that is the EVM ("extended verification module") code, which checks the integrity of files (and their metadata) by checksumming them and comparing the result with a value stored as an extended attribute. The IBAC (integrity-based access control) module can then use EVM and the LSM hooks to allow or deny access to files based on the conclusions reached by the integrity checker. 

All of this can work using a passphrase supplied by the system administrator, but the intended mode of operation uses the trusted platform module (TPM) built into an increasing number of computers. With cooperation from the system's BIOS, the TPM can do an effective job of checksumming the software running on the system. The TPM also performs basic cryptographic functions, like signing the checksums used to verify the integrity of files. The key aspect of the system, though, is that the TPM can be set up to create these signatures only if the checksums for the running system match a set of pre-configured values. The end result is that the checksums associated with files cannot be changed on another system or by booting a different kernel - at least, not in a way which preserves their value as checksums. If the system holds together as advertised, it should be able to prevent attacks based on changing the files used by the system. 

Beyond that, this system supports remote attestation: providing a TPM-signed checksum to a third party which proves that only approved software is running on the system. 

There are clear advantages to a structure like this. A Linux-based teller machine, say, or a voting machine could ensure that it has not been compromised and prove its integrity to the network. Administrators in charge of web servers can use the integrity code in similar ways. In general, integrity management can be a powerful tool for people who want to be sure that the systems they own (or manage) have not be reconfigured into spam servers when they weren't looking. 

The other side of this coin is that integrity management can be a powerful tool for those who wish to maintain control over systems they do not own. Should it be merged, the kernel will come with the tools needed to create a locked-down system out of the box. As these modules get closer to mainline confusion, we may begin to see more people getting worried about them. Quite a few kernel developers may oppose license terms intended to prevent "tivoization," but that doesn't mean they want to actively support that sort of use of their software. Certainly it would be harder to argue against the shipping of locked-down, Linux-based gadgets when the kernel, itself, provides the lockdown tools. 

For now, that issue can be avoided; there are still plenty of more mundane problems with this patch set. But, sooner or later, the integrity management developers are going to get past the lower-level issues; they have certainly shown persistence in working on this patch. Based on his [prior statements](<http://lwn.net/Articles/30048/>), Linus is unlikely to oppose the merging of these modules once they are ready. Whether the rest of the development community will be so welcoming remains to be seen.
