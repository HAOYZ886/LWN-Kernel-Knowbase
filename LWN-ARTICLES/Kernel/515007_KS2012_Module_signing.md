---
title: "KS2012: Module signing"
url: https://lwn.net/Articles/515007/
date: "September 6, 2012"
category: "Modules-Signed"
author: "By Jake Edge September 6, 2012 2012 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
September 6, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

From several accounts, day one of this year's Kernel Summit was largely argument-free. There were plenty of discussions, even minor disagreements, but nothing approaching some of the battles of yore. Day three looked like it might provide an exception to that pattern with a discussion of two different patch sets that are both targeted at cryptographically signing kernel modules. In the end, though, the pattern continued, with an interesting, but tame, session. 

Kernel modules are inserted into the running kernel, so a rogue module could be used to compromise the kernel in ways that are hard to detect. One way to prevent that from happening is to require that kernel modules be cryptographically signed using keys that are explicitly allowed by the administrator. Before loading the module, the kernel can check the signature and refuse to load any that can't be verified. Those modules could come from a distribution or be built with a custom kernel. Since modules can be loaded based on a user action (e.g. attaching a device or using a new network protocol) or come from a third-party (e.g. binary kernel modules), ensuring that only approved modules can be loaded is a commonly requested feature. 

Rusty Russell, who maintains the kernel module subsystem, [called the meeting](<https://lwn.net/Articles/515023/>) to try to determine how to proceed on module signing. David Howells has one [patch set](<https://lwn.net/Articles/511720/>) that is based on what has been in RHEL for some time, while Dmitry Kasatkin [posted](<https://lwn.net/Articles/511502/>) another that uses the digital signature support added to the kernel for integrity management. Howells's patches have been around, in [various forms](<https://lwn.net/Articles/470906/>), since 2004, while Kasatkin's are relatively new. 

Russell prefaced the discussion with an admonishment that he was not interested in discussing the "politics, ethics, or morality" of module signing. He invited anyone who _did_ want to debate those topics to a meeting at 8pm, which was shortly after he had to leave for his plane. The reason we will be signing modules, he said, is because Linus Torvalds wants to be able to sign his modules. 

Kasatkin's approach would put the module signature in the extended attributes (xattrs) of the module file, Russell began, but Kasatkin said that choice was only a convenience. His patches are now independent of the integrity measurement architecture (IMA) and the extended verification module (EVM), both of which use xattrs. He originally used xattrs because of the IMA/EVM origin of the signature code he is using, and he did not want to change the module contents. Since then, he noted a response from Russell to Howells's approach and has changed his patches to add the module signature to the end of the file. 

That led Russell into a bit of a historical journey. The original patches from Howells put the signature into an ELF section in the module file. But, because there was interest in having the same signature on both stripped and unstripped module files, there was a need to skip over some parts of the module file when calculating the hash that goes into the signature. 

The amount of code needed to parse ELF was "concerning", Russell said. Currently, there are some simple sanity checks in the module-loading code, without any checks for malicious code because the belief was that you had to be root to load a module. While that is still true, the advent of things like secure boot and IMA/EVM has made checking for malicious code a priority. But Russell wants to ensure that the code doing that checking is as simple as possible to verify, which was not true when putting module signatures into ELF sections. 

Greg Kroah-Hartman pointed out that you have to do ELF parsing to load the module anyway. There is a difference, though. If the module is being checked for maliciousness, that parsing happens _after_ the signature is checked. Any parsing that is done before that verification is potentially handling untrusted input. 

Russell would rather see the signature appended to the module file in some form. It could be a fixed-length signature block, as suggested by Torvalds, or there could be some kind of "magic string" followed by a signature. That would allow for multiple signatures on a module. Another suggestion was to change the `load_module()` system call so that the signature was passed in, which would "punt" the problem to user space "that I don't maintain anymore", Russell said. 

Russell's suggestion was to just do a simple backward search from the end of the module file to find the magic string, but Howells was not happy with that approach for performance reasons. Instead, Howells added a 5-digit ASCII number for the length of the signature, which Russell found a bit inelegant. Looking for the magic string "doesn't take that long", he said, and module loading is not that performance-critical. 

There were murmurs of discontent in the room about that last statement. There are those who are very sensitive about module loading times because it impacts boot speed. But, Russell said that he could live with ASCII numbers, as long as there was no need to parse ELF sections in the verification code. He does like the fact that modules can be signed in the shell, which is the reason behind the ASCII length value. 

There are Red Hat customers asking for SHA-512 digests signed with 4K RSA keys, Howells said, but that may change down the road. That could make picking a size for a fixed-length signature block difficult. But, as Ted Ts'o pointed out, doing a search for the magic string is in the noise in comparison to doing RSA with 4K keys. The kernel crypto subsystem can use hardware acceleration to make that faster, Howells said. But, Russell was not convinced that the performance impact of searching for the magic string was significant and would like to see some numbers. 

James Bottomley asked where the keys for signing would come from. Howells responded that the kernel build process can create a key. The public part would go into the kernel for verification purposes, while the private part would be used for signing. After the signing is done, that ephemeral private key could be discarded. There is also the option to specify a key pair to use. 

Torvalds said that it was "stupid" to have stripped modules with the same signature as the unstripped versions. The build process should just generate signatures for both. Having logic to skip over various pieces of the module just adds a new attack point. Another alternative is to only generate signatures for the stripped modules as the others are only used for debugging and aren't loaded anyway, so they can be unsigned, he said. Russell agreed, suggesting that the build process could just call out to something to do the signing. 

For binary modules, such as the NVIDIA graphics drivers, users would have to add the NVIDIA public key to the kernel ring, Peter Jones said. 

Kees Cook brought up an issue that is, currently at least, specific to Chrome OS. In Chrome OS, there is a trusted root partition, so knowing the origin of a module would allow those systems to make decisions about whether or not to load them. Right now, the interface doesn't provide that information, so Cook suggested changing the `load_module()` system call (or adding a new one) that passed a file descriptor for the module file. Russell agreed that an additional interface was probably in order to solve that problem. 

In the end, Russell concluded that there was a reasonable amount of agreement about how to approach module signing. He planned to look at the two patch sets, try to find the commonality between the two, and "apply something". In fact, he made a [proposal](<https://lwn.net/Articles/515032/>), based partly on Howells's approach, on September 4. It appends the signature to the module file after a magic string as Russell has been advocating. As he said when wrapping up the discussion, his patch can provide a starting point to solving this longstanding problem.
