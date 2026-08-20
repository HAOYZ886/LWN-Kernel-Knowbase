---
title: Dropping the page cache for filesystems
url: https://lwn.net/Articles/977486/
date: "June 11, 2024"
category: "Filesystems-Page cache"
author: "By Jake Edge June 11, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
June 11, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

VFS maintainer Christian Brauner led a discussion about the possibility of selectively dropping the contents of the page cache for a filesystem in a session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). As he described in his [topic proposal](<https://lwn.net/ml/all/20240116-tagelang-zugnummer-349edd1b5792%40brauner/>), the use case that started him down this path comes from GNOME, which wants to be able to safely suspend access to an encrypted home directory. While it is known to kernel developers, it is surprising to others that reads from encrypted filesystems that have been suspended will succeed if the data to be read still exists in the page cache. 

#### Secrets exposed

Brauner began the session by quickly describing the test case that reproduces the problem, which he included in his proposal (and in his [slides](<https://docs.google.com/presentation/d/1Y42HhE_dHeeOhBUJRN5Lxe1w7XHEFePLmhgzj0zJ-SE/edit?usp=sharing>)). It creates an image file, uses the [LUKS cryptsetup tool](<https://gitlab.com/cryptsetup/cryptsetup>) to turn it into an encrypted volume, makes an XFS filesystem on it, and mounts it. The next step is to write the string "secrets" to a file on the filesystem and use the [luksSuspend](<https://man.archlinux.org/man/cryptsetup-luksSuspend.8.en>) operation to suspend the encrypted block device—a simple `cat` on the file will still show the secret data. 

[ ![\[Christian Brauner\]](https://static.lwn.net/images/2024/lsfmb-brauner-sm.png) ](<https://lwn.net/Articles/977819/>)

He asked if there was some way to add an API that would drop the contents of the page cache so that reading the file would give an error. He went through a list of the objections (and other ideas) that had been posted in the topic-proposal thread, which he had collected onto a slide. 

On the mailing list, Jan Kara had [raised the problem of referenced folios in the page cache](<https://lwn.net/ml/all/20240116114519.jcktectmk2thgagw@quack3>), which could come from a [`vmsplice()`](<https://man7.org/linux/man-pages/man2/vmsplice.2.html>) call; those pages cannot be dropped. Kara also asked about the interaction of a page-cache-drop operation with [`mlock()`](<https://man7.org/linux/man-pages/man2/mlock.2.html>) and [noted](<https://lwn.net/ml/all/20240117143528.idmyeadhf4yzs5ck@quack3>) that pages mapping executable code ""are practically unevictable"". Beyond that, if the intent is to defend against [cold boot attacks](<https://en.wikipedia.org/wiki/Cold_boot_attack>), there will be a [need to zero-out the memory](<https://lwn.net/ml/all/20240215135709.4zmfb7qlerztbq6b@quack3>), since dropping the page-cache entries does not change the memory contents, Kara added. 

Matthew Wilcox had [suggested](<https://lwn.net/ml/all/ZafpsO3XakIekWXx@casper.infradead.org>) using the `security_file_permission()` hook for Linux security modules (LSMs) to inhibit reading the data. So a new BPF LSM program, say, could be attached when the suspend operation is done to return an error for any reads, Brauner said. He thinks there is value in considering some kind of API for page-cache dropping for other use cases, beyond just cold-boot protection. For example, David Howells has told him that he would use an API of that sort for AFS; systemd may also have a use case for it, so it is at least something to consider adding. 

Amir Goldstein said that there are some tests that are part of the [Linux Test Project](<https://github.com/linux-test-project>) (LTP) that currently do "some gymnastics" to try to drop the page cache for a filesystem; it is not reliable, however, so an API would also be useful for testing. Jeff Layton pointed out that Trond Myklebust has posted patches for NFS that will drop the page cache for individual inodes, which might provide part of the solution. Brauner said that Kent Overstreet [mentioned a similar feature](<https://lwn.net/ml/all/h5wq7dsi6r7cjjmkpo2dvn5x662eseluzd2kmzbkzegntzlptd@ncjzyaurmiwb>) for subvolume deletion in bcachefs. Dave Chinner pointed out that [posix_`fadvise()`](<https://man7.org/linux/man-pages/man2/posix_fadvise.2.html>) can drop the page cache for a particular inode already. 

#### Dumb idea?

"You can say it's a dumb idea", Brauner said with a grin. Layton disagreed, saying that it is useful. Wilcox agreed with that: "I can see the use case, it makes sense, it is a reasonable thing to want, it is just a hard thing to implement". There are ways to deal with `vmsplice()` and `mlock()`, he said, by marking those pages as "not up to date" and zeroing them. But it is "really tricky" for things like pages that have been handed off to an RDMA driver, which is maintaining the state of those pages. 

"Is it okay to just fail?", Brauner asked; maybe the operation can just return an error if it cannot be done fully. Wilcox said that probably 99% of the pages could be dropped without encountering one of the corner cases. 

Ted Ts'o said that perhaps part of the problem was that the feature had been introduced as a security feature, which causes developers to immediately focus on the corner cases. [Fscrypt](<https://www.kernel.org/doc/html/v4.18/filesystems/fscrypt.html>) has a "best effort" mechanism that tries to remove all of the directory entries and flush all of the inodes whenever the key gets removed from the keyring. There was no effort to deal with the possibility that encrypted files were still open, because the feature originally targeted ChromeOS, where it only happened after the user session ended. It was not a complete solution and was not advertised as a feature, nor documented. 

That is a possible path forward for this feature, he said, but if there is a need to try to handle all of the corner cases, it is going to be difficult to do. Wilcox said that if you need to ensure that your LibreOffice documents on the filesystem are no longer accessible, that would be fairly straightforward, but if you want to guarantee that for every single file on the filesystem, it is much harder. 

Overstreet said that it was "yet another `revoke()` discussion", which really needs to be resolved at some point. The [`revoke()` system call](<https://lwn.net/Kernel/Index/#revoke>) has been a topic in filesystem circles for more than a decade (at least); it is meant to close all open file descriptors for a given file, but has run aground on a myriad of corner cases. There are multiple places where `revoke()` is needed (`fsck` in corner cases, debugfs), Overstreet said, so someone should tackle the problem. Kara said that he would be reluctant to give guarantees for dropping the page cache, because it greatly depends on the context of how the filesystem is being used, but in many cases it should work fine. 

#### Systemd use

Lennart Poettering said that systemd would like to use this feature, similar to the way that fscrypt does. A best-effort solution would be fine, though he wants to get some kind of error indication when it fails; the filesystem should try to drop as many pages as it can from the cache and, perhaps, report the number that could not be dropped. Systemd will use the control-groups interface to freeze the user session before invoking the operation, which should make for fewer corner-case problems. The systemd use case should not be conflated with the cold-boot-protection case, where the encrypted device needs to be suspended and the pages need to be zeroed out, which is more than what systemd (and others) need, he said. 

Ts'o said that fscrypt reports the number of inodes it was not able to evict. It puts out a log message with that number along with a single inode number as a sample; the information is mostly for ChromeOS developers to use for debugging. It is "half-assed, but that's what we did". 

There is a question of what to do with dirty pages that are in need of writeback when this page-cache removal is done, Chinner said. Are they thrown away, or does the operation wait for the writeback to be done? And where would writeback errors be reported? Brauner said that the belief is that the device has been suspended, so there will be no dirty pages. Ts'o said that when the key gets removed, fscrypt writes all of its dirty inodes, because it will not be able to once the key is gone. If user space has been suspended, as in the systemd use case, there is no one to report any errors to anyway, so a console message is all that can be done. 

Chinner said that it is important to make it clear to users of any new API that they are responsible for ensuring that the filesystem is frozen before dropping its page cache. Dropping the page cache for filesystems that have executable files mapped into memory will not work well, for example. But those are not the types of files people are looking to protect with a feature of this sort, Wilcox said; it is things like document files and web-browser history that people are concerned about. Brauner said that the feature would be targeted at system-level software that is assumed to know what it is doing; Wilcox said that a 99% solution will increase the security for everyone, but it is important not to claim that it is a 100% solution. 

Goldstein wondered about the inode cache; is file data the only concern or should there be protections for metadata as well? Chinner said that trying to purge the inode cache while the filesystem is frozen will deadlock. Wilcox said that the directory-entry cache should also be purged, but did not think that purging the inode cache was needed; Chinner said that could still deadlock, however. 

Ts'o noted that the order used by fscrypt was to get rid of all of the directory entries, shrink the inode cache, do any writeback needed, and, only then, freeze the filesystem and purge the page cache; that is the right order to do those things, but it is all done on a best-effort basis. He suggested that perhaps the luksSuspend operation could add some of these extra steps, since suspending an encrypted device would generally benefit from removing this extra data from memory. 

#### Shrinkers

The current mechanisms available for clearing out the directory-entry and inode caches are the [shrinkers](<https://lwn.net/Kernel/Index/#Memory_management-Shrinkers>), Chinner said, but there is no way to confine them to a single superblock; in addition, those caches are partitioned by memory control groups, so any clearing operation will need to take those into account, as well. Wilcox said that an unmount operation would obviously clear all of this information, so perhaps that could be used. But Chinner said that is not workable for this use case because the users are suspending the system to RAM, and have user-space applications running; an unmount would disrupt all of that. 

Overstreet said that the feature could mesh well with the idea of live filesystem migration. Bcachefs developers are getting ready to release synchronous send and receive for the filesystem, so combining that with a page-cache flush/drop operation would provide the ability to live-migrate a container and its filesystems to another host. Overstreet clarified that he was talking about the same idea as the [Btrfs send and receive](<https://lwn.net/Articles/581558/>). 

There was some discussion of use cases for cleaning up caches on a per-superblock basis, though it is seen as a difficult problem by some. Chinner said that the shrinker interface could be used, with per-superblock functionality added; it would not be hugely difficult, since much of the mechanism for the cleanup is already done there. He has been looking at using shrinkers as a way to implement dropping the caches because the existing ways to do so (using [`/proc/sys/vm/drop_caches`](<https://docs.kernel.org/admin-guide/sysctl/vm.html#drop-caches>)) take a long time. That runs as a single thread, so it can take up to 15 minutes to run for large caches ("with a few hundred million cached inodes"), he said, which is problematic for benchmarking and other uses. 

Kara wondered whether the shrinker was the right interface or whether something based on what is done at unmount time might be better. There is an inherent problem that shrinkers are focused on memory reclaim, rather than on the task needed for this feature. He thought there might be value in a new interface that was somewhat "saner". Chinner acknowledged that there was more to do beyond what the shrinkers do, but thought that they might be part of the solution. 

Luis Chamberlain asked why this new mechanism would not be used whenever a filesystem is frozen. Chinner said that filesystems are often frozen for backup purposes, so the page-cache entries are still important. There was some discussion of the API and whether it should be an [`ioctl()`](<https://www.man7.org/linux/man-pages/man2/ioctl.2.html>) command or something else. Chinner thinks that some new API, probably via a system call, that is not tied to filesystem freezing or anything else, is likely to be the right way forward; that gives the most flexibility if the internals need to change along the way.
