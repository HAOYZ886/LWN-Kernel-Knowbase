---
title: "Case-insensitive ext4"
url: https://lwn.net/Articles/784041/
date: "March 27, 2019"
category: "Filesystems-Case-independent lookups; Filesystems-ext4; UTF-8 encoding"
author: "By Jake Edge March 27, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
March 27, 2019

Handling file names in a case-insensitive way for Linux filesystems has been an ongoing discussion topic for many years. It is a (dubious) feature of filesystems for other operating systems (e.g. Android, Windows, macOS), but Linux has limited support for it. Over the last year or more, Gabriel Krisman Bertazi has been working on the problem for ext4, but it is a messy one to solve. He recently posted his latest patch set, which reflects some changes made at the behest of Linus Torvalds. 

At the 2018 Linux Plumbers Conference (LPC), Krisman [presented his plan](<https://lwn.net/Articles/772960/>) for allowing ext4 filesystems to be case-insensitive. That plan would have enhanced the kernel's Native Language Support (NLS) subsystem to better support multi-byte encodings and expand the case-folding to handle UTF-8. NLS exists to handle filesystems, such as FAT, that support file names with different encodings, which are specified at mount time. Krisman [posted](<https://lwn.net/ml/linux-fsdevel/20181206230903.30011-1-krisman@collabora.com/>) his patch set to make those changes in December shortly after LPC, but Torvalds [objected to the whole idea](<https://lwn.net/ml/linux-fsdevel/CAHk-=wg2JvjXfdZ8K5Tv3vm6+bKRedotF5cr5AwVZVBypVfdAQ@mail.gmail.com/>): 

Why do people want to do this? We know it's a crazy and stupid thing to do. And we know that, exactly because people have done it, and it has always been a mistake. 

He went on to list a number of different problems that can arise with case-insensitivity—many of which have occurred along the way. He asked for use cases: ""I really want to know what is driving this insanity, and what the actual use-case is."" But he made it pretty clear that he was—at a minimum—skeptical. 

The old DOS/Mac people thought case insensitivity was a "helpful" idea, and that was understandable - but wrong - even back in the 80's. They are still living with the end result of that horrendously bad decision decades later. They've _tried_ to fix their bad decisions, and have never been able to (except, apparently, in iOS where somebody finally had a glimmer of a clue). 

Theodore Y. Ts'o, who has been working with Krisman on this effort, had apparently brought the patch set to Torvalds's attention in a private email that Torvalds quotes. Another reply also didn't make it into the thread, but in that message (which Torvalds also [quotes](<https://lwn.net/ml/linux-fsdevel/CAHk-=wgLYy3pRFDxwXB1THf4ev2C6VOmK5m7tfSwwv+EC9pM3Q@mail.gmail.com/>)) Ts'o noted that there was no plan to support encodings other than UTF-8 (and ASCII), which would be set on a per-filesystem basis. Case-insensitivity would be set on a per-directory basis. Given that, Torvalds was adamant that the NLS code was the wrong place to make these changes: 

Either you have a horrible fundamental design mistake that has different per-filesystem locales, or you don't. 

If you don't, you shouldn't be touching any of the nls code. 

Whatever unicode tables you use for case folding shouldn't be in the nls code. 

Ts'o [suggested](<https://lwn.net/ml/linux-fsdevel/20181210000822.GD1840@mit.edu/>) moving the Unicode handling code to `fs/unicode` rather than changing the NLS code. He also described the current state of play with regard to case-sensitivity in filesystems for macOS and Windows, as well as for network filesystems like Samba and NFS. Over time, Ts'o said, the inconsistencies in handling file names between different filesystems have mostly been eliminated. In January, Krisman [posted](<https://lwn.net/ml/linux-fsdevel/20190128213223.31512-1-krisman@collabora.com/>) version 5 of his patch set, which reflects the switch to the `fs/unicode` directory. 

The patch set also makes a more substantial change in that it switches normalization methods. There are multiple ways to create the "same" string in Unicode, which is known as "[equivalence](<https://en.wikipedia.org/wiki/Unicode_equivalence>)". Two different sets of code points that appear the same to a user, but not to the filesystem, would be confusing, so there are [normalization](<https://en.wikipedia.org/wiki/Unicode_equivalence#Normalization>) mechanisms to allow comparisons that take equivalence into account. Ts'o described the confusion that can result: 

In the bad old-days, MacOS X's HFS+ was not normalization-preserving. So it would force filenames to NFD form --- so if the user tried to create a file named Å, and passed in the Unicode string U+212B to creat(2), HFS+ would store it as U+0041,U+030A and that is what readdir(2) would return. Apple has effectively admitted this was a mistake, and their new APFS doesn't do this any more. 

Now, both file systems basically say, "we don't care whether you pass in U+212B or U+0041,U+030A; on the screen it looks identical, Å, so we will treat it as the same filename; but readdir(2) will return what you gave us." 

The new patch set switched from NFKD to NFD, which in normalization lingo means a switch from "compatibility" to "canonical" decomposition: 

The main change presented here is a proposal to migrate the normalization method from NFKD to NFD. After our discussions, and reviewing other operating systems and languages aspects, I am more convinced that canonical decomposition is more viable solution than compatibility decomposition, because it doesn't ignore eliminate any semantic meaning, like the definitive case of superscript numbers. NFD is also the documented method used by HFS+ and APFS, so there is precedent. Notice however, that as far as my research goes, APFS doesn't completely [follow] NFD, and in some cases, like <compat> flags, it actually does NFKD, but not in others (<fraction>), where it applies the canonical form. We take a more consistent approach and always do plain NFD. 

As those quotes indicate, normalization is a messy business. In fact, the whole problem of case handling is a horrific mess, as Torvalds (and others) noted. But there are use cases, mostly involving interoperability with other operating systems. In addition, user-space implementations, with a variety of shortcomings, exist for both Android (to support `/sdcard`) and Samba—those could perhaps be replaced with an in-kernel solution. 

That posting did not generate all that many comments, though there was a [question from Pali Rohár](<https://lwn.net/ml/linux-fsdevel/20190205181041.cdyt5jt7yrqswyy2@pali/>) about the normalization change. He was concerned that NFD would be incompatible with various other Linux user-space tools. But Krisman [explained](<https://lwn.net/ml/linux-fsdevel/8736p2jbov.fsf@collabora.com/>) that the patch set implements name-preserving semantics and that NFD is only used internally for comparison. 

Handling invalid UTF-8 byte sequences also [came up](<https://lwn.net/ml/linux-fsdevel/20190206084752.nwjkeiixjks34vao@pali/>). There are effectively two possible ways to handle the problem, Krisman [said](<https://lwn.net/ml/linux-fsdevel/87sgx0hpiv.fsf@collabora.com/>). Either the filesystem can reject any file name that is invalid UTF-8 (and fix any that are found on the disk) or to simply treat an invalid UTF-8 file name as it would be today, so there would be no case-folding or normalization. Both are implemented and a given filesystem's behavior can be configured with a feature flag; the default is to treat them as an opaque byte sequence as they are currently. 

On March 18, Krisman [posted version 6](<https://lwn.net/ml/linux-fsdevel/20190318202745.5200-1-krisman@collabora.com/>), with few changes from the previous version. He is trying to flush out any opposition to the normalization change (or anything else in the patch set), presumably in the hopes of getting it upstream soon. So far, there has only been a [question from Randy Dunlap](<https://lwn.net/ml/linux-fsdevel/05dfd6a7-49f0-81a7-cd68-ff9f07182461@infradead.org/>) about the impact on ext3 filesystems, which are handled by the ext4 code. Ts'o [noted](<https://lwn.net/ml/linux-fsdevel/20190322235708.GA5675@mit.edu/>) that ""strictly speaking, there is no such thing as an 'ext3 file system'"" these days. Filesystems handled by the ext4 code are defined by the feature bits they have set; if you create a filesystem using "`-t ext3`" and do not override any of the options, though, it will not have any of the new features enabled, thus it will be unaffected by them. 

In order to use the feature, the filesystem will need to be created with encoding-awareness information stored in the superblock. On an encoding-aware ext4 filesystem, case-insensitivity can be enabled on an empty directory (and its children) by setting an inode attribute. That can be done using the `EXT4_CASEFOLD_FL` `ioctl()` command, though eventually the [`chattr`](<http://man7.org/linux/man-pages/man1/chattr.1.html>) command would presumably be updated to add support for the case-folding flag. It should be noted that case-folding and [ext4 encryption](<https://lwn.net/Articles/639427/>) cannot be used concurrently for the same directory, though Krisman is [planning](<https://lwn.net/ml/linux-fsdevel/20190318202745.5200-9-krisman@collabora.com/>) to change that restriction down the road. 

Both encoding-awareness and case-insensitivity are fairly large changes to the traditional handling of file names. Unix file names have always been sequences of any byte values (except NUL and "/") without being interpreted in any way. If these changes are adopted, some ext4 filesystems will now be substantially changing the semantics of various filesystem operations. File creation and renaming will no longer operate the way they do today, for example. 

However, case-insensitivity is a feature that has been a long time coming and we may see it in the mainline before long. At this point, though, it has only run the gauntlet of the filesystem mailing lists; when it gets posted to linux-kernel, there may be others with opinions—or outright objections. If not, though, Linux 5.3 or 5.4 might just have a feature that has been on some people's wish lists for a decade or two. 

`
