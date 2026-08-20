---
title: "A setback for fs-verity"
url: https://lwn.net/Articles/775872/
date: "January 3, 2019"
category: "Filesystems-fs-verity; Security-Integrity verification"
author: "By Jonathan Corbet January 3, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 3, 2019

The [fs-verity mechanism](<https://lwn.net/Articles/763729/>), created to protect files on Android devices from hostile modification by attackers, seemed to be on track for inclusion into the mainline kernel during the current merge window when [the patch set](<https://lwn.net/ml/linux-kernel/20181101225230.88058-1-ebiggers@kernel.org/>) was posted at the beginning of November. Indeed, it wasn't until mid-December that some other developers started to raise objections. The resulting conversation has revealed a deep difference of opinion regarding what makes a good filesystem-related API and may have implications for how similar features are implemented in the future. 

The core idea behind fs-verity is the use of a [Merkle tree](<https://en.wikipedia.org/wiki/Merkle_tree>) to record a hash value associated with every block in a file. Whenever data from a protected file is read, the kernel first verifies the relevant block(s) against the hashes, and only allows the operation to proceed if there is a match. An attacker may find a way to change a critical file, but there is no way to change the Merkle tree after its creation, so any changes made would be immediately detected. In this way, it is hoped, Android systems can be protected against certain kinds of persistent malware attacks. 

There is no opposition to the idea of adding functionality to the kernel to detect hostile modifications to files. It turns out, though, there there is indeed some opposition to how this functionality has been implemented in the current patch set. See the above-linked article and [this documentation patch](<https://lwn.net/ml/linux-kernel/20181101225230.88058-2-ebiggers@kernel.org/>) for details of how fs-verity is meant to work. In short, user space is responsible for the creation of the Merkle tree, which must be surrounded by header structures and carefully placed at the beginning of a block after the end of the file data. An `ioctl()` call tells the kernel that fs-verity is to be invoked on the file; after that, the location of the end of the file (from a user-space point of view) is changed to hide the Merkle tree from user space, and the file itself becomes read-only. 

Christoph Hellwig was the first [to oppose the work](<https://lwn.net/ml/linux-kernel/20181212091406.GA31723@infradead.org/>), less than two weeks before the opening of the merge window. The storage of the Merkle tree inline was, he said, ""simply not acceptable"" and the interface should not require a specific way of storing this data. He later [suggested](<https://lwn.net/ml/linux-kernel/20181213202249.GA3797@infradead.org/>) that the hash data should be passed separately to the `ioctl()` call, rather than being placed after the file data. Darrick Wong [suggested](<https://lwn.net/ml/linux-kernel/20181217200039.GD8111@magnolia/>) a similar interface, noting that it would give the filesystem a lot of flexibility in terms of how the hash data would be stored. 

Dave Chinner [complained](<https://lwn.net/ml/linux-kernel/20181219021953.GD31274@dastard/>) that storing the Merkle tree after the end of the file was incompatible with how some filesystems (XFS in particular) use that space. He [described](<https://lwn.net/ml/linux-kernel/20181219213552.GO6311@dastard/>) the approach as being ""gross"", arguing that it ""bleeds implementation details all over the API"" and creates problems far beyond the filesystems that actually implement fs-verity: 

That's the problem here - fsverity completely redefines the layout of user data files for everyone, not just fsverity, and not just the filesystems that implement fsverity. You've taken an ext4 fsverity implementation feature and promoted it to being a linux-wide file data layout standard that is encoded into the kernel/user ABI/API forever more. 

Chinner, too, argued that the Merkle-tree data should be provided separately to the kernel, rather than being stored in the file itself using a specific format. Filesystem implementations could still put the data after the end of the existing data, but that is a detail that should, according to Chinner be hidden from user space. 

Eric Biggers, the developer of fs-verity, [responded](<https://lwn.net/ml/linux-kernel/20181212202609.GA193967@gmail.com/>) that, while the API requires user space to place the Merkle tree after the end of user data, there is no actual need for filesystems to keep it there: 

As explained in the documentation, the core code uses the "metadata after EOF" format for the API, but not necessarily the on-disk format. I.e., FS_IOC_ENABLE_VERITY requires it, but during the ioctl the filesystem can choose to move the metadata into a different location, such as a file stream. 

He also [said](<https://lwn.net/ml/linux-kernel/20181214044802.GA681@sol.localdomain/>) that passing the Merkle tree in as a memory buffer is problematic, since it could be too large to fit into memory on a small system. (The size of this data also prevents it from being stored as an extended attribute as some have suggested.) Generating the hash data in the kernel was also considered, Biggers said, but it was concluded that this task was better handled in user space. 

Ted Ts'o [claimed](<https://lwn.net/ml/linux-kernel/20181219001603.GD25775@mit.edu/>) repeatedly that there would be no value to be had by changing the API for creating protected files; he described the complaints as ""really more of a philosophical objection than anything else"". The requested API, he said, could be added later (in addition to the proposed API, which would have to be maintained indefinitely) if it turned out to be necessary. After the discussion continued for a while, he [escalated the discussion](<https://lwn.net/ml/linux-kernel/20181221154714.GA26547@mit.edu/>) to Linus Torvalds, asking for a decision: 

Linus --- we're going round and round, and I don't think this is really a technical dispute at this point, but rather an aesthetics one. Will you be willing to accept my pull request for a feature which is being shipped on millions of Android phones, has been out for review for months, and for which, if we *really* need to add uselessly complicated interface later, we can do that? 

**Correction** : I've been reminded that there was [an extensive discussion](<https://marc.info/?l=linux-fsdevel&m=151752635314843>) of this work in early 2018 where many of the same objections were raised. 

Complaining that the code had been out for review makes some sense; it is true that the objections surfaced at something close to the last minute. But that often happens in kernel development; the imminent merging of controversial code can concentrate developers' minds in that direction. Arguing that the API is already being shipped is definitely not a way to win favor. That notwithstanding, Ts'o had clearly hoped for a ruling from Torvalds that the current API was good enough and that the code could be merged. 

What came back might well have failed to please anybody in the discussion, though. It [turns out](<https://lwn.net/ml/linux-kernel/CAHk-=wgdzWgoPSuHeVcqmGE1hB3Gan72r2_AhtC14e60=z45yg@mail.gmail.com/>) that Torvalds has no real objection to the model of storing the hash data at the end of the file itself: 

So honestly, I personally *like* the model of "the file contains its own validation data" model. I think that's the right model, so that you can then basically just do "enable verification on this file, and verify that the root hash is this". 

So that part I like. I think the people who argue for "let's have a separate interface that writes the merkle tree data" are completely wrong. 

From there, though, he made it clear that he was not happy with the current implementation. This model, he said, should be independent of any specific filesystem, so it should be entirely implemented in the virtual filesystem layer. At that point, filesystems like XFS would never even see the fs-verity layer, so its implementation could not be a problem for them. A generic implementation would require no filesystem-specific code and would just work universally. He also disliked the trick that hides the Merkle tree after the fs-verity mode has been set; the validation data for the file should just be a part of the file itself, he said. 

As Ts'o [pointed out](<https://lwn.net/ml/linux-kernel/20181222041712.GC26547@mit.edu/>), keeping the hash data visible in the file would create confusion for higher-level software that has its own ideas about the format of any given file. He also provided [some reasons](<https://lwn.net/ml/linux-kernel/20181223043449.GF26547@mit.edu/>) for why he thinks filesystems need to be aware of fs-verity; they include ensuring that the right thing happens if a filesystem containing protected files is mounted by an older version of the filesystem code. Making fs-verity fully generic would, he said, have forced low-level API changes that would have affected ""dozens of filesystems"", a cost that he doesn't think is justified by the benefits. 

The last message from Ts'o was sent on December 22; Torvalds has not responded to it. ~~There has not, however, been a pull request for fs-verity sent, and it is getting late in the merge window for such a thing to show up.~~ [**Correction** : [a pull request](<https://patchwork.kernel.org/patch/10745561/>) was sent copied only to the linux-fscrypt mailing list; it has not received a response as of this writing.] It seems likely that fs-verity is going to have to skip this development cycle while the patches are reworked to address some of the objections that have been raised — those from Torvalds, at least. Even then, the work might be controversial; it is rare for the kernel to interpret the contents of files, rather than just serving as a container for them, and some developers are likely to dislike an implementation that depends on that sort of interpretation. But if Torvalds remains in favor of such an approach, it is likely to find its way into the kernel eventually.
