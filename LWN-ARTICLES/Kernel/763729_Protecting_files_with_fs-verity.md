---
title: "Protecting files with fs-verity"
url: https://lwn.net/Articles/763729/
date: "August 30, 2018"
category: "Filesystems-fs-verity; Security-Integrity verification"
author: "By Jonathan Corbet August 30, 2018"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 30, 2018

The developers of the Android system have, among their many goals, the wish to better protect Android devices against persistent compromise. It is bad if a device is taken over by an attacker; it's worse if it remains compromised even after a reboot. Numerous mechanisms for ensuring the integrity of installed system files have been proposed and implemented over the years. But it seems there is always room for one more; to fill that space, the [fs-verity](<https://lwn.net/ml/linux-fsdevel/20180824161642.1144-1-ebiggers@kernel.org/>) mechanism is being proposed as a way to protect individual files from malicious modification. 

The core idea behind fs-verity is the generation of a [Merkle tree](<https://en.wikipedia.org/wiki/Merkle_tree>) containing hashes of the blocks of a file to be protected. Whenever a page of that file is read from storage, the kernel ensures that the hash of the page in question matches the hash in the tree. Checking hashes this way has a number of advantages. Opening a file is fast, since the entire contents of the file need not be hashed at open time. If only a small portion of the file is read, the kernel never has to bother reading and checking the rest. It is also possible to catch modifications made to the file after it has been opened, which will not be caught if the hash is checked at open time. 

One could imagine a number of implementations where the kernel would create and maintain the Merkle tree, and protect it from being modified in its own right. The actual implementation in fs-verity is not such a scheme, though; much of the work is pushed out to user space. Developers wanting to know how to work with fs-verity will be needing to know how to do this work; for now we are told that ""A documentation file in Documentation/filesystems/ is planned but not yet included"". So, for now, one must reverse-engineer the patch set instead. 

The process starts with the creation of a file to be protected in the normal way. User space must then make a pass over the file to generate the Merkle tree so that it can append a series of structures to the file: 

  * The contents of the Merkle tree itself. 
  * An `fsverity_descriptor` structure (defined at the end of [the first patch](<https://lwn.net/ml/linux-fsdevel/20180824161642.1144-2-ebiggers@kernel.org/>) in the series) containing information about the fs-verity version in use, the hash algorithm employed, etc. It also includes a count of "extension" structures to follow containing additional information. 
  * The series of extension structures. One of these, for example, contains a signed copy of the root hash of the Merkle tree; this can be used to verify that the tree, itself, has not been tampered with. 
  * An `fsverity_footer` structure at the end of the file; it only contains a magic number and the offset of the `fsverity_descriptor` structure. 

Once user space has populated the file with this information, it must reopen the file read-only, then perform an `ioctl()` call with the `FS_IOC_ENABLE_VERITY` command (which takes no arguments). A number of interesting things happen at that point, starting with the fact that the file is rendered immutable; it can be deleted, but the contents of the file can no longer be changed. The underlying filesystem tweaks the visible length of the file so that it appears to end _before_ the Merkle tree, thus hiding the fs-verity metadata. And, of course, it will also cause the fs-verity checking to be performed for any subsequent accesses; any attempt to read a page with the wrong hash will fail with an `EIO` error. 

This is the first public posting of the fs-verity patch set, but the idea has been circulating for a while; it was first [described](<https://lwn.net/Articles/763795/>) in January, and was [discussed](<https://lwn.net/Articles/752614/>) at the 2018 Linux Storage, Filesystem, and Memory-Management Summit. While the idea of providing integrity guarantees at the file level has some appeal, there are a number of persistent concerns about this particular approach to the problem. 

One that was extensively discussed in January is the overlap with the kernel's integrity measurement architecture (IMA) mechanism, which already exists to ensure the integrity of individual files. There are a number of reasons for avoiding IMA in this setting, it seems. IMA itself is seen as a large and complex mechanism that also requires cooperation from the security-module subsystem; the Android developers are looking to avoid this complexity. IMA hashes an entire file at open time, making opens expensive (especially when only a small part of the file is read) and making it unable to catch changes made after the file has been opened. And IMA is intended to protect a system starting at boot time, but Android has its own boot-time mechanisms. 

The fact that security modules are not involved in fs-verity decisions at all was an area of disagreement in its own right. IMA developer Mimi Zohar [argued](<https://lwn.net/Articles/763838/>) that, while fs-verity could check for file integrity, the policy decision of what to do about a verification failure should be implemented in a security module. Ted Ts'o [pointed out](<https://lwn.net/Articles/763796/>) that not all kernel security mechanisms work with security modules, and maintained that fs-verity should not either. 

While it has not been implemented, the fs-verity developers have mentioned the possibility of interoperating with IMA in the future. In particular, the fs-verity hashes could be used by IMA to verify files without having to read their entire contents. 

The "hide the Merkle tree after the end of the file" technique has raised some concerns of its own. It exposes the details of the mechanism to user space, requiring that it will be supported forever and making it hard to change. Copying a file (to a backup, for example) will lose the verity information. Having the file be longer than its stated length may confuse code in both the kernel and user space, and may be hard for some filesystems to implement; the current patch set has support for ext4 and f2fs. This mechanism will also work poorly for network filesystems like NFS, which would like to have better integrity support but which cannot access the hidden Merkle tree. 

There does not appear to be a viable alternative to this mechanism at the moment, though. One often-suggested idea is to put the Merkle tree in an extended attribute, but the amount of space available for extended attributes is not enough to hold the tree for larger files. It would be possible to simply not store the lower levels of the tree and to regenerate them when the file is opened — Merkle trees support working in that mode — but that would add complexity and make opens more expensive. 

The fs-verity developers also defend the current approach by noting the advantages of storing the tree as ordinary file data. The tree, which can indeed become large, can be managed in the page cache in the usual way. If the underlying filesystem is encrypted, the fs-verity metadata will also be encrypted with no additional effort. Thus, fs-verity developer Eric Biggers [argued](<https://lwn.net/ml/linux-fsdevel/20180825041647.GA726@sol.localdomain/>), the best way to avoid the after-end-of-file hack would be to finally support named file streams in the kernel. That idea spawned [an extensive thread of its own](<https://lwn.net/ml/linux-fsdevel/20180825135107.GA12251@bombadil.infradead.org/>) that makes interesting reading for people interested in the details; the executive summary is that the developers who have looked at the problem are mostly united in thinking that it is not a good idea. 

The end result of all this is that there do not seem to be a lot of changes in store for this patch set. Nobody has — yet — put a foot down and expressed outright opposition to it being merged in its current form. Unless that changes, something closely resembling the current patch set seems likely to find its way into the mainline fairly soon. 

(See also: [slides [PDF]](<https://events.linuxfoundation.org/wp-content/uploads/2017/11/fs-verify_Mike-Halcrow_Eric-Biggers.pdf>) from the recent Linux Security Summit session on fs-verity.)
