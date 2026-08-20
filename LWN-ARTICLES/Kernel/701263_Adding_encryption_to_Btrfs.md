---
title: Adding encryption to Btrfs
url: https://lwn.net/Articles/701263/
date: "September 21, 2016"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet September 21, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 21, 2016

One of the promises of the Btrfs filesystem is that its new design would facilitate the addition of modern features like compression and encryption. Compression has been there for a while, but Btrfs has yet to gain support for encryption; indeed, the ext4 filesystem [got this feature first](<https://lwn.net/Articles/639427/>) over a year ago with an implementation that is also used by the f2fs filesystem. Work to fill this gap is underway, as can be seen in [this recently posted patch set](<https://lwn.net/Articles/700487/>) from Anand Jain, but it would appear that encryption in Btrfs remains a distant goal. 

It remains distant because it has become clear that this code will not be merged in anything like its current form. With luck, though, it should be the source of a lot of lessons that can be applied to later, hopefully more successful attempts. Sometimes, one simply has to stumble a few times when attacking a difficult problem space. 

#### Crypto troubles

There is an aspect to cryptographic code development that has been learned the hard way many times over: this code needs to be written with help from people who understand cryptography well and know where the pitfalls are. Developers who set out without that domain knowledge are certain to make serious mistakes. So this is not a good way to introduce an encryption-related patch set: 

Also would like to mention that a review from the security experts is due, which is important and I believe those review comments can be accommodated without major changes from here. 

As Dave Chinner (among others) [pointed out](<https://lwn.net/Articles/701266/>), it is far too late for a security review, which should really happen during the design phase. The ext4 encryption feature, he noted, did go through a design review phase ahead of the posting of any code, and quite a bit of useful feedback was the result. 

In this case, it would appear that this kind of review would have been helpful. Eric Biggers, who is working on the ext4 encryption feature, [looked at the code](<https://lwn.net/Articles/701267/>) and came back with a harsh judgment: 

You will also not get a proper review without a proper design document which details things like the threat model and the security properties provided. But I did take a short look at the code anyway because I was interested. The results were not pretty. As far as I can see the current proposal is fatally flawed as it does not provide confidentiality of file contents against a basic attack. 

Alex Elsayed also [pointed out](<https://lwn.net/Articles/701269/>) some of the cryptographic problems in the code. It comes down to a poor choice of encryption modes that leaves a filesystem open to well-understood known-plaintext attacks. The reviewers said that a mode like [XTS](<https://en.wikipedia.org/wiki/Disk_encryption_theory#XTS>), which lacks this particular vulnerability, should have been used instead. Or, even better, an [authenticated encryption (AE)](<https://en.wikipedia.org/wiki/Authenticated_encryption>) approach should be used; AE modes are believed to be far more resistant to most known attacks. AE brings its own challenges, though; the (mostly obsolete) ecryptfs filesystem uses it, but the current ext4/f2fs implementation does not. A related issue, as Ted Ts'o [pointed out](<https://lwn.net/Articles/701275/>), is the increasing importance of taking advantage of hardware-based encryption for performance; that will tend to rule out ""exotic encryption modes"" in favor of something boring (but hardware-supported) ~~like AES~~. 

#### Crypto at the wrong level?

Another criticism of the patch set is that it implements a Btrfs-specific encryption infrastructure, rather than using the generic infrastructure added at the virtual filesystem (VFS) layer and used by ext4 and f2fs. One motivation for that approach is that Btrfs encryption is managed at the [subvolume](<https://lwn.net/Articles/579009/>) level, meaning that a single master key is used for the entire subvolume. Ext4 and f2fs, instead, lack the subvolume concept; they provide file-level encryption that allows different users to have different keys within the same filesystem. Another result is that Btrfs does not benefit from the work that has been done on the VFS infrastructure; as Chinner put it: 

The generic file encryption code is solid, reviewed, tested and already widely deployed via two separate filesystems. There is a much wider pool of developers who will maintain it, review changes and know all the traps that a new implementation might fall into. There's a much bigger safety net here, which significantly lowers the risk of zero-day fatal flaws in a new implementation and of flaws in future modifications and enhancements. 

He compared Btrfs-specific encryption to the Btrfs RAID5/6 implementation, which has had known problems for years and appears to be essentially unmaintained. ""Encryption simply cannot be treated like this - it has to be right, and it has to be well maintained."" Some Btrfs developers bristled at the description of the filesystem's RAID implementation, but there was general agreement that the VFS code should be used to the greatest extent possible — and improved in places where it cannot yet be used. 

Btrfs does provide some unique challenges that will stress the capabilities of the existing VFS code. That code, for example, manages encryption keys as an inode attribute; that is how file-level encryption is supported. Btrfs throws a spanner into that works in a couple of ways: 

  * If Btrfs snapshots are present, an inode is likely to be present in more than one of them. Without a great deal of care, these snapshots could be used to force a reuse of the encryption keys and "nonce" values used with a specific file; many AE algorithms will fail catastrophically if that happens. 

  * In general, Btrfs does a lot of sharing of file blocks at the extent level. That is how the copy-on-write mechanism works in general, and features like [deduplication](<https://lwn.net/Articles/679031/>) will cause even more sharing to happen. Once again, this sharing could be used to expose encrypted traffic, or to simply tell when one party has modified a file that shares extents with another. 

A solution to some of these problems would be to simply copy extents and do without the sharing when encryption is involved. But another solution falls out of the requirements: encryption in Btrfs probably needs to be managed at the extent level, rather than at the file level. That would reduce the potential for nonce-reuse attacks and would eliminate problems that would otherwise result if one file sharing an extent is modified in a way that changes the extent's offset within the file. 

As Btrfs developer Zygo Blaxell [put it](<https://lwn.net/Articles/701272/>), the Btrfs extent-use model already creates challenges for the VFS layer: 

Currently any extent in the filesystem can be shared by any inode in the filesystem (assuming the two inodes have compatible attributes, which could include encryption policy), including multiple references from the same inode to the same extent at different logical offsets. This is the basis of the deduplication and copy_file_range features. 

This confuses the VFS caching layer when dealing with deduped reflinked, or snapshotted files. It's not surprising that VFS crypto has problems coping with it as well. 

At the moment, encryption at the VFS level doesn't have any real concept of extents at all; extents are generally something that only specific filesystems know about. So the VFS file-encryption code is not suitable for solving the Btrfs encryption problem in its current form. As many have pointed out, though, the solution is not to start over, but to enhance the VFS code to get it to the point where it can do the job. 

About the only definite conclusion that came from the discussion was that there is still a lot of work to do before the Btrfs encryption problem is even well understood, much less properly implemented. If nothing else, the patches posted so far have served as a focus point for a discussion that needs to happen and, hopefully, a starting point for the next try, sometime in the future. Once again, we see that cryptography is hard, and the intersection with a next-generation filesystem makes it even harder.
