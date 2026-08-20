---
title: Making O_TMPFILE atomic (and statx() additions)
url: https://lwn.net/Articles/896153/
date: "May 25, 2022"
category: "Filesystems-statx; O TMPFILE"
author: "By Jake Edge May 25, 2022 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 25, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Right on the heels of his [previous filesystem session](<https://lwn.net/Articles/896055/>) at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Steve French led a session on temporary files and their interaction with network filesystems. The problem is that creating temporary files is not always atomic, so he was proposing changing that, which would eliminate a possible race condition and be more efficient for network filesystems. Since the temporary-file discussion did not fill the 30-minute slot, however, French took the opportunity to discuss some attributes he would like to see get added for the [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) system call. 

Calling [`open()`](<https://man7.org/linux/man-pages/man2/open.2.html>) with the `O_TMPFILE` flag creates a unnamed file that, by default, is deleted when it is closed. It is not a feature that was in Linux from the outset; it was [added for the 3.11 kernel](<https://lwn.net/Articles/562294/>) in 2013. Not all filesystems implement the functionality, but the most widely used ones do. There are two types of filesystems, he said, some that have a two-step process for creating a file and others that do it in one step. In the two-step case, the file is created and then, separately, opened, while the others do both of those things in a single step. 

When those operations are performed for a network filesystem like SMB, there is a problem. If there are two operations to create the temporary file, the network filesystem has to do something special or the file created will be removed before the open can occur. For some filesystems, the create operation returns an open file, which is normally closed when the create operation completes. But if the file created is a temporary file, the close will, of course, delete the file. In that case, that close operation that would normally be done at the end of the create step has to be deferred so that the open operation can succeed. 

There is a small possibility of a race between the create and open operations, but it is also inefficient to make two calls across the network when one should suffice, he said. Combining the two operations, similar to what [`atomic_open()`](<https://elixir.bootlin.com/linux/latest/source/fs/namei.c#L3192>) does, would be a better approach. He suggested adding a directory inode operation called `atomic_tmpfile()` that filesystems could implement if they want to support the feature. 

David Howells wondered if it made sense to simply use `atomic_open()` and add code to it for the temporary-file case. French said he looked at that and it is possible to do it that way, but that raises an issue that he would like to discuss at next year's LSFMM. He said that the open and create paths in the virtual filesystem (VFS) code are "kind of ugly" and confusing. Beyond that, there are places where unnecessary stat operations are being performed, which causes a costly network round-trip for network filesystems. So he sees some cleanup that he thinks needs to be done in those code paths. 

Christian Brauner said that it would be better, if possible, to make the change for atomic temporary files at the VFS level so that all filesystems could benefit without needing to add code. French thought that sounded like a good idea, but Howells was concerned that some filesystems might not be able to support the atomic temporary-file creation, so VFS might not be the right place. Forcing filesystems to open the temporary file at the same time they create it might be problematic for, say, overlayfs, he said. It is worth experimenting with the idea, French said. 

#### `statx()`

Since there was time left in the slot, French shifted gears to talk about another idea he would like to see implemented. There are already a number of flags that are returned by `statx()`, he said, but he can see a need for a few more. He put up a slide listing nine attribute flags that currently can be returned for a file, but there are four additional attributes that "jump out at me" for addition, he said. 

For example, it is relatively common these days for people to have "local" files that are actually stored in the cloud somewhere, so an "offline" attribute would be useful. On the flipside, a "pinned" attribute could be used to indicate a file that is backed by cloud storage but is hosted locally, so it should not be removed because of the time required to get it back. These are not attributes that network filesystems, such as SMB, would need to handle, they would simply report them. These "seem like no-brainers", he said. 

The other two are "integrity" and its opposite, to indicate some kind of scratch file where file integrity is not important, which he called "no scrub" on his slides. These would ask the filesystem to either do the best it can in terms of integrity protection or to do nothing in that regard. Chuck Lever questioned whether a single bit is enough to encompass all of the complexity of Linux integrity protection, which has various configuration options and policies. But `statx()` already has "encrypted" and "compressed" attributes, so French sees "integrity" in the same light; it would be requesting the strongest integrity protection the filesystem can provide. 

Howells wondered which of these attribute bits would actually be used by applications. Putting them into `statx()` implies that applications will use them frequently. He can see that "offline" might make sense, since it would provide a useful hint to desktop environments, but the others seem questionable. The filesystem may need to know about them, but it is less clear that applications need them. 

Ted Ts'o said that he was hearing an assumption that there is a way to set these attributes, but that is not the case. `statx()` only reports them and there is no Linux system call that would allow an administrator to set them. The attribute flags originated in an ext2-specific `ioctl()` command, he said, that eventually got adopted by other filesystems and moved into the VFS. But the original 32-bit flag field was the actual on-disk representation for the ext filesystems so there were ext-specific flags that other filesystems were not interested in. 

`statx()` came about to report a filesystem-independent set of attributes to user space. But there is no way for someone to change the value of those bits in a filesystem-independent way. There are various mechanisms to set them, using `ioctl()` commands, but no system call to set, for example, the `statx()` "integrity" attribute for any filesystem. 

There was some discussion of what a "setinfo" facility might look like. Kent Overstreet suggested that the extended attribute (xattr) interface could be used; a special namespace would actually refer to these file attributes and `statx()` would be the fast path to access them. French thought that sounded reasonable, and did not think it was urgent to add the ability to set the values in a generic way.
