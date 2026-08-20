---
title: Caching for extended attributes
url: https://lwn.net/Articles/1074919/
date: "June 2, 2026"
category: "Filesystems-Extended attributes"
author: "By Jake Edge June 2, 2026 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
June 2, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

[Extended attributes](<https://man7.org/linux/man-pages/man7/xattr.7.html>) (xattrs) provide a way to attach key/value metadata to inodes—files, directories, and the like—in a filesystem. As with many Linux filesystems, the [FUSE filesystem](<https://docs.kernel.org/filesystems/fuse/>) supports xattrs. In a filesystem-track session at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), FUSE maintainer Miklos Szeredi led a discussion about caching xattrs in kernel memory; he would like to create some common infrastructure that could be used by FUSE and shared with other filesystems. 

He began by listing the existing xattr-cache implementations in the kernel. There is a [`simple_xattr`](<https://elixir.bootlin.com/linux/v7.0.10/source/include/linux/xattr.h#L107>) cache that is used by tmpfs, pidfs, and kernfs; it uses a hash table to store the data but it does not allow any shrinking. The [`nfs4_xattr_cache`](<https://elixir.bootlin.com/linux/v7.0.10/source/fs/nfs/nfs42xattr.c#L19>) also uses a hash table to store the xattrs; it does allow shrinking. The [`mb_cache`](<https://elixir.bootlin.com/linux/v7.0.10/source/fs/mbcache.c#L11>) is used by ext2 and ext4 to deduplicate xattrs and is not a general-use xattr cache. Meanwhile, the [EA inode](<https://docs.kernel.org/next/filesystems/ext4/eainode.html>) feature is used by ext4 to store large xattrs that do not fit into the inode. Ted Ts'o cautioned that the EA inode was not intended as a cache at all, just as a way to store large xattrs, though it could perhaps be used as one. Amir Goldstein pointed out that the XFS buffer cache effectively acts as an xattr cache, which Szeredi acknowledged. 

[ ![\[Miklos Szeredi\]](https://static.lwn.net/images/2026/lsfmb-szeredi-sm.png) ](<https://lwn.net/Articles/1075406/>)

If there is to be a common implementation of the cache, it should probably use a hash table, Szeredi said. The basic idea of consolidating the two main xattr-cache implementations (`simple_xattr` and `nfs4_xattr_cache`) into a common code base that could then be used for FUSE seemed good to attendees. Currently FUSE requires a round-trip through user space to retrieve xattrs, so caching those that are used frequently will help its performance. 

He went through some of the requirements for the shared xattr cache, including having low overhead for storing only a few xattrs and even lower overhead when not storing any. There is a need to be able to reclaim memory, especially that taken up by large xattr values; the NFS xattr cache has a complicated mechanism to do so already, he said. 

Brauner asked about access-control list (ACL) caching, which is already done for filesystems. The [`i_acl` field](<https://elixir.bootlin.com/linux/v7.0.10/source/include/linux/fs.h#L771>) in the [`inode` structure](<https://elixir.bootlin.com/linux/v7.0.10/source/include/linux/fs.h#L761>) is used for that, he said, and wondered if that caching would also be part of this new shared scheme. Szeredi pointed out that `i_acl` is not a cache, it is simply an entry for the ACL for the inode; its use would be unaffected by the shared xattr cache. 

Continuing on, Szeredi said that the xattr cache might globally deduplicate the keys and values in order to save memory. One use case for that, Ts'o said, is ACLs for Windows that are stored as xattrs; they can be ""gargantuan"", and are generally shared by every file in a directory. Deduplicating those would save substantial amounts of memory; that is also true for POSIX ACLs, he said. 

Brauner asked what was meant by "global" and wondered if Szeredi meant per superblock. Szeredi said he was not yet sure about the granularity of the deduplication. Ts'o recommended doing it on a per-superblock basis in part because that would ease memory accounting for control groups; the xattr-cache use for a particular control group could be charged correctly. 

There are some policy questions that go along with the cache, Ts'o said. Administrators may want to limit the size of xattrs that are added to the cache. For example, caching large xattrs, rather than more important ones like the `security.*` xattrs, may not be desired. Szeredi thought that restricting the cache to only store certain types of xattrs might be a way to do that. 

There was also some hard-to-follow discussion on the implementation, but the overall feeling in the room was positive toward the idea. There were no objections to trying to find some common ground, rather than adding yet another xattr cache specifically for FUSE. 

[I would like to apologize for any errors here. The acoustics in the room were problematic for both hearing and recording. Misunderstanding and misidentification may have resulted.]
