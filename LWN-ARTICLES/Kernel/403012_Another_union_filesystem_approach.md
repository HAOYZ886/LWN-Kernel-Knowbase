---
title: Another union filesystem approach
url: https://lwn.net/Articles/403012/
date: "September 1, 2010"
category: "Filesystems-Union; Overlayfs"
author: "By Jake Edge September 1, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
September 1, 2010

Creating a union of two (or more) filesystems is a commonly requested feature for Linux that has never made it into the mainline. Various implementations have been tried ([part 1](<http://lwn.net/Articles/325369/>) and [part 2](<http://lwn.net/Articles/327738/>) of Valerie Aurora's look from early 2009), but none has crossed the threshold for inclusion. Of late, [union mounts](<http://lwn.net/Articles/396020/>) have been making some progress, but there is still work to do there. A hybrid approach--incorporating both filesystem- and VFS-based techniques--has recently been posted in an [RFC patchset](<http://lwn.net/Articles/402276/>) by Miklos Szeredi. 

The idea behind unioning filesystems is quite simple, but the devil is in the details. In a union, one filesystem is mounted "atop" another, with the contents of both filesystems appearing to be in a single filesystem encompassing both. Changes made to the filesystem are reflected in the "upper" filesystem, and the "lower" filesystem is treated as read-only. One common use case is to have a filesystem on read-only media (e.g. CD) but allow users to make changes by writing to the upper filesystem stored on read-write media (e.g. flash or disk). 

There are a number of [details](<http://lwn.net/Articles/324291/>) that bedevil developers of unions, however, including various problems with namespace handling, dealing with deleted files and directories, the POSIX definition of `readdir()`, and so on. None of them are insurmountable, but they are difficult, and it is even harder to implement them in a way that doesn't run afoul of the technical complaints of the VFS maintainers. 

Szeredi's approach blends the filesystem-based implementations, like unionfs and aufs, with the VFS-based implementation of union mounts. For file objects, an `open()` is forwarded to whichever of the two underlying filesystems contains it, while directories are handled by the union filesystem layer. Neil Brown's very helpful first cut at [documentation](<https://lwn.net/Articles/403038/>) for the patches lumped directory handling in with files, but Szeredi [called that a bug](<https://lwn.net/Articles/403049/>). Directory access is never forwarded to the other filesystems and directories need to ""come from the union itself for various reasons"", he said. 

As outlined in Brown's document, most of the action for unions takes place in directories. For one thing, it is more accurate to look at the feature as unioning directory trees, rather than filesystems, as there is no requirement that the two trees reside in separate filesystems. In theory, the lower tree could even be another union, but the current implementation precludes that. 

The filesystem used by the upper tree needs to support the "trusted" extended attributes (xattrs) and it must also provide valid `d_type` (file type) for `readdir()` responses, which precludes NFS. Whiteouts--that is files that exist in the lower tree, but have been removed in the upper--are handled using the "trusted.union.whiteout" xattr. Similarly, opaque directories, which do not allow entries in the lower tree to "show through", are handled with the "trusted.union.opaque" xattr. 

Directory entries are merged with fairly straightforward rules: if there are entries in both the upper and lower layers with the same name, the upper always takes precedence unless both are directories. In that case, a directory in the union is created that merges the entries from each. The initial mount creates a merged directory of the roots of the upper and lower directory trees and subsequent lookups follow the rules, creating merged directories that get cached in the union dentry as needed. 

Write access to lower layer files is handled by the traditional "copy up" approach. So, opening a lower file for write or changing its metadata will cause the file to be copied to the upper tree. That may require creating any intervening directories if the file is several levels down in a directory tree on the lower layer. Once that's done, though, the hybrid union filesystem has little further interaction with the file, at least directly, because operations and handed off to the upper filesystem. 

The patchset is relatively small, and makes very few small changes to VFS--except for a change to `struct inode_operations` that ripples through the filesystem tree. The `permissions()` member of that structure currently takes a `struct inode *`, but the hybrid union filesystem needs to be able to access the filesystem-specific data (`d_fsdata`) that is stored in the dentry, so it was changed to take a `struct dentry *` instead. David P. Quigley [questioned](<https://lwn.net/Articles/403070/>) the need for the change, noting that unionfs and aufs did not require it. Aurora [pointed out](<https://lwn.net/Articles/403071/>) that union mounts would require something similar and that, along with Brown's documentation, seemed to put the matter to rest. 

The rest of the patches make minor changes. The first adds a new `struct file_operations` member called `open_other()` that is used to forward `open()` calls to the upper or lower layers as appropriate. Another allows filesystems to set a `FS_RENAME_SELF_ALLOW` flag so that `rename()` will still process renames on the identical dummy inodes that the filesystem uses for non-directories. The bulk of the code (modulo the `permissions()` change) is the new [`fs/union` filesystem](<https://lwn.net/Articles/403081/>) itself. 

While "union" tends to be used for these kinds of filesystems (or mounts), Brown [noted](<https://lwn.net/Articles/403084/>) that it is confusing and not really accurate, suggesting that "overlay" be used in its place. Szeredi is not opposed to that, [saying](<https://lwn.net/Articles/403085/>) that "overlayfs" might make more sense. Aurora more or less [concurred](<https://lwn.net/Articles/403086/>), saying that union mounts were called "writable overlays" for one release. The confusion stemming from multiple uses of "union" in existing patches (unionfs, union mounts) may provide additional reason to rename the hybrid union filesystem to overlayfs. 

The `readdir()` semantics are a bit different for the hybrid union as well. Changes to merged directories while they are being read will not appear in the entries returned by `readdir()`, and offsets returned from `telldir()` may not return to the same location in a merged directory on subsequent directory opens. The lists of directory entries in merged directories are created and cached on the first `readdir()` call, with offsets assigned sequentially as they are read. For the most part, these changes are ""unlikely to be noticed by many programs"", as Brown's documentation says. 

A bigger issue is one that all union implementations struggle with: how to handle changes to either layer that are done outside of the context of the union. If users or administrators directly change the underlying filesystems, there are a number of ugly corner cases. Making the lower filesystem be read-only is an attractive solution, but it is non-trivial to enforce, especially for filesystems like NFS. 

Szeredi would like to define the problem away or find some way to enforce the requirements that unioning imposes: 

The easiest way out of this mess might simply be to enforce exclusive modification to the underlying filesystems on a local level, same as the union mount strategy. For NFS and other remote filesystems we either 

a) add some way to enforce it,  
b) live with the consequences if not enforced on the system level, or  
c) disallow them to be part of the union. 

There was some discussion of the problem, without much in the way of conclusions other than a requirement that changing the trees out from under the union filesystem not cause deadlocks or panics. 

In some ways, hybrid union seems a simpler approach than union mounts. Whether it can pass muster with Al Viro and other filesystem maintainers remains to be seen however. One way or another, though, some kind of solution to the lack of an overlay/union filesystem in the mainline seems to be getting closer.
