---
title: The two sides of reflink()
url: https://lwn.net/Articles/331808/
date: "May 5, 2009"
category: Filesystems; System calls; reflink
author: "By Jonathan Corbet May 5, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 5, 2009

One of the discussions your editor missed at the recent Linux Storage and Filesystem workshop covered the proposed `reflink()` system call. Fortunately, the filesystem developers have now filled in the relevant information with a detailed email exchange, complete with patches. We now have a [proposed system call](<http://lwn.net/Articles/331576/>) which has created more open questions than answers. The creation of a new core system call requires a lot of thought, so a close look at these questions would seem to be called for. 

The proposed system calls are pretty simple: 

```
int reflink(const char *oldname, const char *newname);
    
        int reflinkat(int old_dir_fd, const char *oldname,
                      int new_dir_fd, const char *newname, int flags);
```

These system calls function much like `link()` and `linkat()` but with an important exception: rather than create a new link pointing to an existing inode, they create a new inode which happens to share the same disk blocks as the existing file. So, at the conclusion of a `reflink()` call, `newname` looks very much like a copy of `oldname`, but the actual data blocks have not been duplicated. The files are copy-on-write, though, meaning that a write to either file will cause some or all of the blocks to be duplicated. A change to one of the files will thus not be visible in the other file. In a sense, a `reflink()` call behaves like a low-cost file copy operation, though how copy-like it will be remains to be seen. 

The first question to arise was: does the kernel really need to provide both the `reflink()` and `reflinkat()` system calls? Most of the other *`at()` calls are paired with the non-at versions because the latter came first. Since Unix-like systems have had `link()` for a long time, it cannot be removed without breaking applications. So `linkat()` had to go in as a separate call. But `reflink()` is new, so it can just as easily be implemented in the C library as a wrapper around `reflinkat()`. That is how things will probably be done in the end. 

The deeper discussion, though, reveals that there are two fundamentally different views of how this system call should work. Joel Becker, who posted the `reflink()` patch, sees it as a new variant of the `link()` system call. Others, though, would like it to behave more like a file copy operation. If you see `reflink()` as being a type of `link()`, then certain implications emerge: 

  * The reflink-as-link view requires that the new file have (to the greatest extent possible) the same metadata as the old one; in particular, it must have (at the end of the `reflink()` system call) the same owner, just like hard links do. 

  * Creating low-level snapshots of filesystems (or portions thereof) is straightforward and easy. Reflinked files look just like the originals; in particular, they have (mostly) the same metadata and can share the same security context. 

  * Disk quotas are a problem. Should a reflinked file count against the owner's disk quota, even though little or no extra storage is actually used? If so, the system must take extra steps to keep users from creating a reflink to a file they do not own; otherwise one user could exhaust another user's quota. If, instead, storage is charged against the quota of the user who created the reflink, complicated structures will be needed to track usage associated with files owned by others. 

  * What happens if the new file's metadata - permissions or owner - are changed? In some scenarios, depending on the underlying filesystem implementation, it seems that a metadata change could require a copy-on-write of the whole file. That would turn a command like `chmod` into a rather heavy-weight operation. 

On the other hand, if a reflink is like making a copy, the situation changes somewhat: 

  * Security becomes a rather more complicated affair. Making a hard link doesn't require messing with SELinux security contexts, but a reflink-as-copy would require that. Permission checks (again, including security module checks) would have to become more elaborate; it would have to be clear that the user making the reflink had read access to the file. 

  * The quota problem goes away. If a reflink is essentially a copy, then the resulting link should be owned by the user who creates it, rather than the owner of the original file. The only course which makes sense is to charge both users for the full size of the file. There are no concerns about one user exhausting another's disk quota, and there are no real difficulties with keeping disk usage information current. 

  * Metadata changes are handled naturally, since the files are completely separate from each other. 

  * Reflinks are no longer true snapshots; they will not work to represent the state of the filesystem at a given time. For a user whose real interest is low-level snapshotting, reflink-as-copy will not work. 

On the other hand, reflink-as-copy could be used in a lot of other interesting situations; the `cp` command could create reflinks by default when the destination file is on the same filesystem. That would turn "`cp -r`" into a fast and efficient operation. They could also be used to share files between virtualized guests. 

What it comes down to is that there are real uses for both the reflink-as-link and reflink-as-copy modes of operation. So the right solution may well be to implement both modes. The `flags` parameter to `reflinkat()` can be used to distinguish between the two. Implementing both behaviors will complicate the implementation somewhat, and it muddies up what is otherwise a conceptually clean system call. But that's what happens, sometimes, when designs encounter the real world.
