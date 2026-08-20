---
title: "Filesystems and case-insensitivity"
url: https://lwn.net/Articles/772960/
date: "November 28, 2018"
category: "Filesystems-Case-independent lookups"
author: "By Jake Edge November 28, 2018 Linux Plumbers Conference"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
November 28, 2018

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2018-Linux_Plumbers_Conference>)

A recurring topic in filesystem-developer circles is on handling case-insensitive file names. Filesystems for other operating systems do so but, by and large, Linux filesystems do not. In the Kernel Summit track of the 2018 [Linux Plumbers Conference](<https://linuxplumbersconf.org/>) (LPC), Gabriel Krisman Bertazi described his plans for making Linux filesystems encoding-aware as part of an effort to make ext4, and possibly other filesystems, interoperable with case-insensitivity in Android, Windows, and macOS. 

Case-insensitive file names for Linux have been discussed for a long time. The oldest reference he could find was from 2002, but it has come up at several [Linux Storage, Filesystem, and Memory-Management Summits](<https://lwn.net/Archives/ConferenceIndex/#Storage_Filesystem__Memory_Management>) (LSFMM), including [in 2016](<https://lwn.net/Articles/685431/>) and in Krisman's [presentation this year](<https://lwn.net/Articles/754508/>). It has languished so long without a real solution because the problem has many corner cases and it is "tricky to get it right". 

[ ![\[Gabriel Krisman Bertazi\]](https://static.lwn.net/images/2018/lpc-krisman-sm.jpg) ](<https://lwn.net/Articles/772971/>)

An attendee asked about XFS and its handling of case-insensitive file names. Krisman said that when an XFS filesystem is created, it can be configured to handle them. It is ASCII-only, though a [proposal](<https://www.spinics.net/lists/xfs/msg30069.html>) from SGI in 2014 would have added full UTF-8 support for XFS and extended the case-handling to Unicode file names. 

The traditional Unix approach is that file names are opaque byte sequences that cannot contain "/" characters. He is proposing to add encoding awareness to filesystems, but, he asked, what are the advantages of doing so? For one thing, Windows and macOS have encoding-aware filesystems; it is a feature that Linux lacks. There are "real world use cases" as well: porting from the Windows world, dealing with the case-insensitive tree that Android exposes, and, in general, providing better support for exported filesystems. Android has a user-space hack for case handling, but it is slow and has many race conditions. An encoding-aware filesystem is a better way to expose this functionality to users, he said. 

Unicode can represent the "same" string in multiple different ways, via composition for example, but that is confusing. Multiple files with the same-appearing name in a directory, as he showed in his [slides [PDF]](<https://linuxplumbersconf.org/event/2/contributions/242/attachments/140/173/case-insensitive-lookups.pdf>), will be difficult to deal with. That means some kind of normalization will need to be applied. Beyond that, "case" is really only defined in terms of an encoding—it is meaningless for a byte sequence. That is why he implemented encoding awareness before tackling case insensitivity. 

The kernel has a Native Language Support (NLS) subsystem but it has multiple limitations. It has trouble dealing with invalid character sequences—in some situations it returns zero, in others something else. It can't deal with multi-byte sequences or code points; for example, `to_upper()` and `to_lower()` return a single byte. There is no support for dealing with the evolution of encodings, which is not really a problem for UTF-8 except for unmapped code points—case folding for unmapped points is not stable, he said. In addition, NLS is missing support for normalization and has only partly implemented case folding; the latter is "almost ASCII only". 

#### Start with NLS

So he has been proposing improvements to NLS as part of his [encoding and case-insensitive support patch set](<https://lwn.net/ml/linux-ext4/20181017205524.23360-1-krisman@collabora.co.uk/>) that has been posted to the ext4 mailing list. It provides a new `load_nls_version()` function that allows the caller to define the encoding and version that it wants to use. It has a flags argument that allows filesystems to specify the normalization type, case-fold type, and permissiveness mode they want. That version and behavior information would be stored in the superblock of the filesystem. 

Krisman's changes would add support for multi-byte characters by adding a new API for comparisons, normalization, and case folding. It will support UTF-8 [NFKD](<https://en.wikipedia.org/wiki/Unicode_equivalence#Normal_forms>) normalization that is based on code from the 2014 SGI patch set. It uses a decoding [trie](<https://en.wikipedia.org/wiki/Trie>) and the mechanism is extendable to other normalization types. For example, if support for the Apple filesystem was needed, NFD normalization could be added. The changes he is making are all backward compatible with existing NLS tables and users, Krisman said. 

He currently has patches for the kernel, e2fsprogs, and xfstests out for review. This effort is quite different from what he presented at LSFMM back in May. 

There was some discussion among attendees about the changes. The original file name will be preserved when it is created, Krisman said, so that makes the filesystem "case preserving" like NTFS. Concern was expressed about containers sharing a filesystem with encoded file names, but having different user-space encodings. That is not a use case that is envisioned, he said; root filesystems will not normally be encoding aware. The most common use cases, Ted Ts'o said, are USB sticks with a FAT filesystem that does case folding or users of other operating systems accessing the filesystem through Samba. A storage appliance will be able to create a case-folding filesystem and Samba can turn off its expensive user-space case-handling solution. 

Another use case that Krisman brought up was for [SteamOS](<https://store.steampowered.com/steamos/>), which would have a separate partition for game data that would be encoding aware. Ts'o said that there are some inherent assumptions in this work. The primary users will be like the SteamOS or Samba appliance examples and that "all the world is UTF-8". It would be hugely complicated to support different directories with different encodings, he said. He invited those present to point out any problems they see with those assumptions. 

James Bottomley asked if the user-space side had been consulted on these choices. He noted that European distributions typically use single-byte encodings and that the Chinese hate UTF-8 because all characters become four bytes in size. Ts'o said that the problem is essentially being handed off to the distributions. POSIX does not have a way for filesystems to communicate the encoding of their file names; if that existed, glibc could handle the differences. 

There is no good solution for that problem, Ts'o continued. There will be information in the superblock, which should be exposed via [`statfs()`](<http://man7.org/linux/man-pages/man2/statfs.2.html>). That will take some time to happen, so perhaps a sysfs field could be used in the interim. 

Krisman said that his implementation tries to make good use of the directory entry (dentry) cache. Equivalent names do not create multiple dentries, there is just one per file. The `d_hash()` and `d_compare()` routines needed to be made encoding aware. For now, negative dentries (asserting the _absence_ of a given file name) are not cached; it will require some work to carefully invalidate negative dentries during file creation. 

#### On to case-insensitivity

Supporting case-insensitive file names requires the encoding-awareness changes in order to define what case folding means for a given character. A per-directory inode attribute can be set to turn on case-insensitivity, but that is only allowed on empty directories to avoid name collisions. Case-insensitivity is trivial to implement once the encoding support is available, he said; it is effectively just a special case of encoding. 

There are some limitations of the current implementation, starting with the lack of negative dentries in the cache. Directory encryption is not supported since the lookup is based on the hash of the name, but the same hash cannot be generated from two names that normalize to the same name. He proposed storing the file using the hash of the normalization, but was not sure if that would solve the problem. 

Another problem area is how to deal with invalid byte sequences. He proposes falling back to the previous behavior, just treating the names as sequences of bytes, when a sequence is invalid for the encoding. There may be some user-space breakage due to normalization or case folding of file names that will need to be handled as well. 

The current implementation is for the ext4 filesystem, but the main part is the NLS changes. The ext4-specific changes give other filesystems a roadmap to adding encoding-awareness and case-insensitivity, Krisman said. Ts'o noted that there is no active NLS maintainer currently, so he will take Krisman's changes through the ext4 tree. He will try to test other users of NLS, but explicitly is not volunteering to take on NLS maintenance going forward. 

Boaz Harrosh pointed out that Linus Torvalds called negative dentries important for performance reasons. He wondered if there were plans to add them for encoding-aware filesystems. Krisman said that invalidating negative dentries needs careful thought and code but that it should be doable. The path for file renames is particularly tricky. Bottomley asked why negative dentries needed to be handled differently than positive ones. The problem is that many people want case-preserving filesystems, so looking up `FOO` when `foo` exists should generate a negative dentry for `FOO` but that will interfere with case-insensitive lookups for `Foo` or even `foo`. 

The reaction to this proposal was much more positive than to Krisman's earlier attempt. It would seem that we will soon have the ability to handle case-insensitive ext4 filesystems and the potential is there to add it for others. 

[I would like to thank LWN's travel sponsor, The Linux Foundation, for assistance in traveling to Vancouver for LPC.]
