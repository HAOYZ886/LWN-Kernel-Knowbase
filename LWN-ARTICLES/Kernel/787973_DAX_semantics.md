---
title: DAX semantics
url: https://lwn.net/Articles/787973/
date: "May 13, 2019"
category: "DAX; Memory management-Nonvolatile memory"
author: "By Jake Edge May 13, 2019 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 13, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

In the filesystems track at the 2019 Linux Storage, Filesystem, and Memory-Management Summit, Ted Ts'o led a discussion about an inode flag to indicate DAX files, which is meant to be applied to files that should be directly accessed without going through the page cache. XFS has such a flag, but ext4 and other filesystems do not. The semantics of what the flag would mean are not clear to Ts'o (and probably others), so the intent of the discussion was to try to nail those down. 

Dan Williams said that the XFS DAX flag is silently ignored if the device is not DAX capable. Otherwise, the file must be accessed with DAX. Ts'o said there are lots of questions about what turning on or off a DAX flag might mean; does it matter whether there are already pages in the page cache, for example. He said that he did not have any strong preference but thought that all filesystems should stick with one interpretation. 

While Christoph Hellwig described things as "all broken", Ts'o was hoping that some agreement could be reached among the disparate ideas of what a DAX flag would mean. A few people think there should be no flag and that it should all be determined automatically, but most think the flag is useful. He suggested starting with something "super conservative", such as only being able to set the flag for zero-length files or only empty directories where the files in it would inherit the flag. Those constraints could be relaxed later if there was a need. 

[ ![\[Ted Ts'o\]](https://static.lwn.net/images/2019/lsf-tso-sm.jpg) ](<https://lwn.net/Articles/788291/>)

Boaz Harrosh wondered why someone might want to turn DAX off for a persistent memory device. Hellwig said that the performance "could suck"; Williams noted that the page cache could be useful for some applications as well. Jan Kara pointed out that reads from persistent memory are close to DRAM speed, but that writes are not; the page cache could be helpful for frequent writes. Applications need to change to fully take advantage of DAX, Williams said; part of the promise of adding a flag is that users can do DAX on smaller granularities than a full filesystem. 

When he developed DAX, he added the DAX flag as a "chicken bit", Matthew Wilcox said. The intent was that administrators could control the use of DAX on their systems; he would like to preserve that ability going forward. It may not only depend on whether the application is DAX aware or not, but also on the workload that the application is handling. Ts'o said that there may be applications that want to use persistent memory in its "full unexpurgated form"; requiring administrators to set a flag on a file to enable that is not particularly friendly. Wilcox agreed, saying that he did not want to make administrator's jobs harder, but he did want to preserve the ability to override what an application developer chose. 

But Chuck Lever wondered why these DAX-aware applications would want to use a filesystem at all; wouldn't they rather simply get access to a chunk of persistent memory directly via `mmap()` or similar? Williams said that is exactly what DAX does; if you `mmap()` a DAX file, you get a chunk of the persistent memory mapped in. The problem is that other applications using the same filesystem may not be ready to get that same kind of access; they may be relying on filesystem semantics for file-backed memory. 

One way to look at it, Ts'o said, is that if someone buys a chunk of persistent memory for a single application that is going to use all of it, they don't need a filesystem at all. They can just point the application directly at the block device. But if someone wants to share that persistent memory with multiple applications, user IDs, and such, then a filesystem makes sense. 

Lever asked why some kind of namespace would not be used to make that distinction. Williams said that administrators are used to the tools to deal with filesystems and files, so that would make a better interface. For the "I know what I want" users, device DAX solves their problems, but for other users, some other interface is needed and file-oriented interfaces make the most sense. In addition, Ts'o pointed out that a namespace or bind mount is not sufficient since a file either needs to use the page cache or it needs to stay completely out of it, trying to do both will lead to problems. Partitioning a block device into DAX and non-DAX portions would be another way to make it all work, but that lacks flexibility. 

An attendee asked what it meant for an application to be DAX aware. Williams said that DAX-aware applications want to do in-place updates of their data and want to manage the CPU cache themselves; these applications want to do accesses on data that is smaller than a page in size directly in memory versus having some kind of buffering or the page cache. 

Ts'o said that there are a number of papers out there that describe libraries that can use persistent memory to, for example, update a B-Tree in place. These algorithms do the operations in the right order and flush the caches in such a way that a crash at any point will leave the data structure in a consistent state. It is important to note that these are libraries that would be used by applications because Ts'o said he would not normally trust application authors to get this kind of thing right. But Hellwig expressed skepticism that any non-academic filesystem author would actually trust the CPU's memory subsystem to always get this right; that was met with a fair amount of laughter. 

Amir Goldstein asked how allowing DAX and non-DAX access to a file was different from allowing buffered and direct I/O access to the same file. Hellwig said that, while mixing buffered and direct I/O is allowed, it does not give the results that users expect. Goldstein also asked about mixing direct I/O and DAX, but that does not work, Kara said; mixing buffered and direct I/O doesn't really work either, but the kernel pretends that it does. For DAX, kernel developers decided not to repeat that mistake. 

Direct I/O and DAX do not work together, but they could if someone wanted to rework the existing implementation, Hellwig said. It would be useful to be able to read persistent memory without using the page cache and to write via the page cache, but it would be tremendously complex to handle the CPU cache coherency correctly, which is probably what has scared everyone away. Another problem is when a DAX-aware application gets surprised by having the page cache between it and persistence; it believes that the cache-flush instructions it issues are making the data persistent when they are not. 

But Hellwig said that there is an API problem if applications are issuing "random weird instructions" and expecting them to work; there are too many other layers potentially in between the application and the storage. He is not entirely sure that making these kinds of programming models more widespread is a good idea, but if that's the path that will be taken, there should be some kind of interface at the VDSO level that applications can call where the kernel will ensure that they do the right thing. The kernel will issue the proper cache-flush instructions or whatever else is necessary. The existing model for applications "can't ever work", he said. 

Ts'o said that there can be a debate about how reliable these models and cache-flush instructions truly are, but he is reasonably confident that if they don't work, the hardware vendors will fix them when customers put pressure on them. In any case, though, that is orthogonal to the question of having a per-file DAX flag and what its semantics should be. 

Williams said he was uncomfortable calling it a "DAX flag", though he acknowledged that could lead directly to the bike shed. He thought that perhaps a `MAP_SYNC` flag on the inode would be better. Hellwig suggested that the name "DAX" should be retired because it is confusing at this point; he, of course, had his own suggestion for a name for the flag ("writethrough"), as did several others, though no real conclusion was reached. 

The discussion moved onto how the flag, however named, could be set. Ts'o said he was uncomfortable restricting it to empty directories, and then having all of the files in it inheriting the attribute, due to hard links and renames. If that is really going to be the way forward, then filesystems need to look at restricting hard links and renames. This is why he wants to nail down the semantics before implementing it in filesystems. 

Lever is uncomfortable with a "permanent sticky bit" in the filesystem that is set by administrators, however. He is concerned that administrators will turn it off when it needs to be on, or the reverse; he wondered if a flag to `open()` was a better path, since the application should know what it needs. But Hellwig pointed out that `open()` flags are not checked to see if unsupported options are being passed; applications could not be sure to get the behavior they asked for. 

Williams pointed out that there already is a `dax` mount option, so that ship has already sailed to some extent. Ts'o also noted that `open()` is not the right time to specify this; it needs to be a property of the file itself. If the file is opened twice, once with DAX and once without, what would that mean? One way to handle that might be to fail an `open()` with the "wrong" mode if the file is already open; the "real disaster" for buffered versus direct I/O was in allowing both types of `open()` to succeed. Beyond that, though, Hellwig was emphatic that `open()` flags should never be used for data integrity purposes. 

Sprinkled throughout the latter part of the discussion were more suggestions of different names for the flag, but Ts'o thinks they are stuck with the DAX name. There were also questions about how per-file flags interact with the global mount option, including whether a `nodax` mount option was required. Most seemed to think that option was not needed, however. 

In summary, Ts'o said that he thought the overall consensus was to have a flag for empty directories that would be inherited on files created there. The flag could also be set for zero-length files and he heard no enthusiasm for allowing the flag to be cleared once it was set. He plans to summarize the discussion in a post to the relevant mailing lists (fsdevel and DAX) for further discussion.
