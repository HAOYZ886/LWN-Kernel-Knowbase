---
title: "File-level integrity"
url: https://lwn.net/Articles/752614/
date: "April 27, 2018"
category: "Filesystems-fs-verity; Security-Integrity verification"
author: "By Jake Edge April 27, 2018 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 27, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At the 2018 Linux Storage, Filesystem, and Memory Management Summit, Ted Ts'o introduced an integrity feature akin to [dm-verity](<https://lwn.net/Articles/459420/>) that targets Android, at least to start with. It is meant to protect the integrity of files on the system so that any tampering would be detectable. The initial use case would be for a certain special type of Android file, but other systems may find uses for it as well. 

Android has a system partition that is read-only and protected with dm-verity. It must be completely rewritten in order to update it; after that, a reboot is required to start using the new data. Updating a filesystem protected with dm-verity is a heavyweight operation, Ts'o said. But Android has some system-level programs that need to be updated with some frequency, so there is the idea of privileged Android packages (APKs) that would not live in the system partition. These are somewhat like setuid-root binaries, Ts'o said, but Google wants them to be updated like any other app—in the background, possibly unnoticed by the user. 

Normally, APKs have a signature that is checked once at download time and then never checked again. For the privileged APKs, Google wants to do the signature check on every use of the APK. That sounds like a job for the [integrity measurement architecture](<https://lwn.net/Articles/488906/>) (IMA), which targets file-level integrity and is already in the kernel, but its performance is not really up to the Android use case, Ts'o said. APKs can be large, with multiple translations and other pieces that are never used. Doing a checksum of the entire package before executing it will slow down the user experience and use more power; fs-verity will only check the pieces of the APK as they are actually needed. 

The way it does that is by creating a file-level [Merkle tree](<https://en.wikipedia.org/wiki/Merkle_tree>) that has a cryptographic hash for each page-sized block of the file. The root of the tree will be signed; verifying hashes as the tree is traversed is then enough to ensure that those parts of the file have not been changed. 

These APKs will be marked as immutable in the filesystem; they will need to be replaced whenever they are updated. The Merkle tree will be placed directly after the normal file data and the tree will be followed by a header that will store the size (`i_size`) of the original APK file. That will be used as the size of the file when it is accessed by the rest of the system. 

Reporting a smaller `i_size` is perhaps the most controversial part of fs-verity, Ts'o said. For an immutable file, though, he doesn't think it will cause problems elsewhere. He considered using dm-verity with loopback mounts, but that would require all APK-handling code throughout the system to be updated, while fs-verity with the `i_size` switch allows the verification to be transparent to the rest of the system. 

Bruce Fields asked about performance versus IMA, but Ts'o has not measured it; he believes it will be a big win for low-powered ARM devices, though. The key used by fs-verity would either be baked into the kernel directly or there would be a key-signing key in the kernel that would be used to verify the key used to sign the Merkle tree. It is the same basic model as for signed kernel modules, he said. 

Jan Kara asked about accessing data beyond `i_size`, noting that calling [`mpage_getpages()`](<https://elixir.bootlin.com/linux/latest/source/fs/mpage.c#L318>) will not work. Ts'o acknowledged that and said that fs-verity has its own scheme for reading pages past `i_size` and populating the page cache. Right now, there are "some hacks" that will need to be cleaned up before the code can go upstream. 

Chris Mason wondered if that mechanism could be generalized to provide file streams (or forks) for Linux. Ts'o said that would mean that other filesystems beyond just ext4 would need to implement it. Mason argued that this feature is already adding stream support, just for a single, specialized type. But Dave Chinner noted that there is already some precedent for filesystems storing data beyond the reported `i_size`: XFS directories do so. Ts'o also pointed out that he is able to do a bunch of simplification in the code because these files are immutable. 

The initial implementation of fs-verity is "going to be massively cheating", Ts'o said; the code will be found in the Android kernel repositories, but that is not the code that will be proposed for the mainline. He has been talking with Mimi Zohar about integrating fs-verity with IMA and he plans to discuss its design at the Linux Security Summit.
