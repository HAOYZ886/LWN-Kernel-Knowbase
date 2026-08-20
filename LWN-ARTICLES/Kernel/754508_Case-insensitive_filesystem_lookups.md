---
title: "Case-insensitive filesystem lookups"
url: https://lwn.net/Articles/754508/
date: "May 23, 2018"
category: "Filesystems-Case-independent lookups"
author: "By Jake Edge May 23, 2018 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 23, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Case-insensitive file name lookups are a feature that is fairly frequently raised at the Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). At the 2018 summit, Gabriel Krisman Bertazi proposed a new way to support the feature, though it met with a rather skeptical reception—with one notable exception. Ted Ts'o seemed favorably disposed to the idea, in part because it would potentially be a way to get rid of some longstanding Android ugliness: [wrapfs](<https://lwn.net/Articles/718640/>). 

Krisman noted that proposals for case-insensitive lookups show up on the Linux kernel mailing list periodically. He has incorporated some of that work into his proposal, including some [SGI patches from 2014](<https://www.spinics.net/lists/xfs/msg30069.html>) that implemented Unicode support and case-folding for XFS. His [patches](<https://www.spinics.net/lists/linux-ext4/msg59577.html>) would add support to the VFS layer with filesystem-specific hooks to actually do the insensitive hashes and lookups. 

[ ![\[Gabriel Krisman Bertazi\]](https://static.lwn.net/images/2018/lsf-bertazi-sm.jpg) ](<https://lwn.net/Articles/754536/>)

The intent is to be able to bind mount a subtree of a case-sensitive filesystem in a case-insensitive form, but there are changes needed to the directory entry cache to make it all work. David Howells asked if there would potentially be different case-folding functions for each mount point and Krisman indicated there would be. 

The Android use case is to support the `/sdcard` directory (which was FAT-based and thus case-insensitive in the early days) on ext4, which is case-sensitive, of course. Ts'o said that it is a legacy Android feature that is "kind of ugly", but he would like to get rid of the out-of-tree hacks that are currently being used to support it. It is "legacy insanity", Dave Chinner said. The bind-mount approach being proposed is the "least insane way to implement what is, I grant, a somewhat insane thing", Ts'o said; Android apps expect that behavior and breaking user space is something that the Android project avoids. 

Al Viro wanted to know how Krisman's code would handle two different directory-cache entries (dentries) that had the "same" name but with different case for some letters. Krisman said there would be a hash function that would hash to the same value for names that differ only by case, so there would be only one dentry per case-insensitive name. The exact name with its case preserved would be stored in the dentry, though. 

A problem comes from negative dentries: assertions that a given file name does not exist, which are cached when a lookup fails. He is proposing a "hard negative dentry" that would assert that there is no file that would satisfy a case-insensitive lookup. If the filesystem determines that there should be a hard negative dentry for a given name, it would invalidate all but one of the negative dentries for any other case variants. 

But if there is the ability to have both `foo` and `FOO` on the disk, which would a lookup return on the case-insensitive side? That and a number of other problems with what Krisman is suggesting were discussed in a rapid-fire round-robin that was difficult to capture. The general upshot was that most in the room were fairly skeptical of the approach. 

Ts'o said that for the Android case, which is much of the reason for the "case-sensitive and case-insensitive view of the same subtree" use case, 99% of the time the access is through the case-insensitive view. There are, however, certain ways to access those files in a case-sensitive way and some apps are dependent on this behavior. It would be more sane to have case-sensitivity be a property of the directory, but it is not clear to him how much that would break apps in the wild. He could start that conversation with the Android team, he said. 

Chinner said that what XFS has done for case-insensitive file names "may look stupid and slow", but it is much faster than what Samba is doing. However, it requires marking a filesystem as being case-insensitive at `mkfs` time. A case-insensitive hash is used for the dentries and there can be no names in the filesystem that only differ by case. 

Krisman concluded by noting that his code is mostly working at this point, though there are still some problems. In particular, there are difficulties with collisions of positive dentries. Returning the first dentry found is unpredictable, so it will return an exact match if it can, but if there are multiple entries and no exact match, it is not clear what to return.
