---
title: Filtering fanotify events with BPF
url: https://lwn.net/Articles/1018493/
date: "May 6, 2025"
category: BPF
author: "By Daroc Alden May 6, 2025 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Daroc Alden**  
May 6, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Linux systems can have large filesystems; trying to keep up with the stream of [ fanotify](<https://www.kernel.org/doc/html/latest/admin-guide/filesystem-monitoring.html>) filesystem-monitoring notifications for them can be a struggle. Fanotify is one of a few ways to monitor accesses to filesystems provided by the kernel. Song Liu led a discussion on how to improve in-kernel filtering of fanotify events to a joint session of the filesystem and BPF tracks at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. He wants to combine the best parts of a few different approaches to efficiently filter filesystem events. 

There are two ways to monitor and restrict filesystem actions on Linux, Liu said: fanotify and Linux security modules (LSMs). They both have benefits and drawbacks. The main problem with using LSM hooks to respond to filesystem events is that LSM hooks are global — the LSM must respond to accesses for all files, even if it's only interested in a subset of files. The main problem with fanotify is that notifications are handled in user space, incurring a lot of context switches. The best of both worlds would be to have efficient mask-based filtering for relevant files (like fanotify) and fast in-kernel handling for the more complicated cases (like LSMs). 

One member of the audience pointed out that LSM hooks are invoked for all filesystem operations, but fanotify can only block calls to [ `open()`](<https://man7.org/linux/man-pages/man2/open.2.html>) and [ `read()`](<https://man7.org/linux/man-pages/man2/read.2.html>), so they're not really comparable. Liu agreed, but said that was a separate topic. 

[ ![\[Song Liu\]](https://static.lwn.net/images/2025/song-liu-lsfsmmbpf-small-2.png) ](<https://lwn.net/Articles/1018490#second>)

Liu then went into a little more detail about how [ BPF-LSM hooks](<https://docs.kernel.org/bpf/prog_lsm.html>) work. Multiple BPF programs can attach to the `bpf_lsm_file_open()` hook. When a file is opened, the kernel will iterate through each different rule to see whether it wants to block the `open()`. Most BPF programs don't apply to most files, Liu said, but they are still run. Fanotify filters, on the other hand, use a bitmask to quickly eliminate events that the filter is not interested in. 

There are problems with combining the approaches, though. Currently, of all the different types of BPF program, only BPF-LSM programs have storage associated with an inode object. Filesystem developers generally want to avoid bloating the inode structure too much, which means that extra storage for LSMs lives in a separate structure: `inode->i_security`. Like many parts of the kernel's design, this is a tradeoff; for example, it allows the use of read-copy-update (RCU) protections, but it also adds an additional dereference to access the information and it prevents other types of BPF program from accessing the data. 

To solve this, Liu wants to move the data currently stored in the `i_security` blob into the inode structure (conditionally, based on the value of a kernel-configuration setting). That will make the data available to more BPF program types, and therefore to his prototype solution for fanotify BPF filtering. That suggestion didn't sit well with Josef Bacik, though, who pointed out that the inode structure is embedded in other places, such as in some Btrfs structures. So changing its size has the potential to change cache-line alignment or, more importantly, change the number of inodes that can fit in a page. Liu agreed that could be a problem. 

Amir Goldstein asked what kinds of data Liu expected to be stored in the inode. He replied that it depended on the use case, but that one example might be cached information about whether access to a file should be allowed. 

One current problem with fanotify is that it is not "local", Liu explained. There's a long chain of functions called from [ `fsnotify_open()`](<https://elixir.bootlin.com/linux/v6.14.4/source/include/linux/fsnotify.h#L459>) to decide whether the operation will be allowed. In particular, the check for whether a superblock has any watchers is fairly cumbersome. If the fast mask-based check could be moved earlier, it might improve performance. 

Goldstein thought that made sense, but pointed out that the fanotify mask is the combined mask of all watchers, so it would make sense to have LSMs use the same mask just to indicate that ""someone is interested"". Liu agreed. The kernel should optimize for the case when nobody is watching, he said, so that the LSM hook doesn't have to be called for every file. 

Unfortunately, the simple idea of having a mask that can be checked quickly and is shared between LSMs and fanotify watches is complicated by subtree monitoring. Liu summarized the existing options for handling watches on subtrees as: add a separate mask entry to everything in the subtree individually, walk up the filesystem tree for each operation, or match the full path of the directory against a pattern. None of those options are great. The first option makes applying and removing watches potentially slow for large directories. The second option makes file accesses slower, and the last option requires a good deal of complexity to find the full path of a file. 

In Liu's patch set, he took the approach of setting a fanotify mark for a whole filesystem, and then using [ `is_subdir()`](<https://elixir.bootlin.com/linux/v6.14.4/source/fs/dcache.c#L3064>), which checks whether one entry is a subdirectory of another, to filter events further in BPF. That required letting BPF programs call `is_subdir()`, which seems straightforward, except that it lets programs hold references to directory entries in BPF maps, which in turn prevents the filesystem from being unmounted. Bacik questioned why holding onto directory entries was necessary here, and Liu explained that he wanted to use them to quickly check whether something was in the subtree of interest. 

Jeff Layton pointed out that since directories can't be hard-linked, the BPF program could hold onto an inode for the directory instead. Liu agreed that if it were possible to quickly check for membership in a subtree using an inode, that would work for his purposes. Goldstein thought that it could make sense to use [ fsnotify](<https://lwn.net/Articles/339399/>) to set a mark on a directory, and let that code handle things. 

Christian Brauner thought that checking whether something was in a subtree was not quite as simple as Layton and Goldstein made it sound because of the global rename lock. Goldstein asked whether `is_subdir()` really took a lock, to which Brauner replied that it did. He went on to explain that until recently there had been an issue where `is_subdir()` would retry when it detected a concurrent rename, but that apparently people had deep subdirectories that could cause `is_subdir()` to stall when there were only a few rename operations, so it was changed to acquire the lock. He agreed to go over the code with Goldstein later, and informed Liu that unless he was willing to deal with false positives, any solution was going to contend on the rename lock. 

Goldstein suggested that the operation could ignore the lock entirely and keep calling `get_parent()` — theoretically, it could loop forever, but only if someone were continuously relocating the directories being traversed. Brauner didn't think this was sufficient for a Linux _security_ module, where correctness is especially important. Liu indicated that there were several cases where security-critical code, such as the [ Landlock LSM](<https://docs.kernel.org/security/landlock.html>), does something like this, and asked what a better solution would look like. Goldstein suggested an approach similar to [ audit](<https://github.com/linux-audit/audit-documentation/wiki>): track renames, so that cached information can be invalidated on rename. 

Several other participants indicated that they had their own suggestions, but at that point the session ran out of time, and discussion moved to the hallway. It seems clear that the general idea of Liu's proposal, allowing the creation of BPF LSMs that don't need to be invoked for irrelevant file accesses, had plenty of support. The specifics of the design, however, remain up in the air.
