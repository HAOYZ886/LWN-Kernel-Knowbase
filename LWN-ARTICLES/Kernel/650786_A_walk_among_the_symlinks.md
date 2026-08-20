---
title: A walk among the symlinks
url: https://lwn.net/Articles/650786/
date: "July 14, 2015"
category: "Filesystems-Virtual filesystem layer"
author: "July 14, 2015 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 14, 2015

This article was contributed by Neil Brown

This is the third and final article in a series on pathname lookup in the Linux kernel; the first two were [an introduction to pathname lookup](<https://lwn.net/Articles/649115/>) and [a look at RCU-walk](<https://lwn.net/Articles/649729/>). Thus far, the discussion has carefully avoided the complex subject of symbolic links, but that is about to change. Linux 4.2 will contain a substantial rewrite of much of the code for handling symbolic links in pathname lookup, which was part of the motivation for writing this series. Now we finally have enough background understanding to explore how this new symlink handling works. 

Symbolic links were first introduced into Unix with 4.1c-BSD in the early 1980s, but were not uniformly heralded as a good idea. Questions arose concerning whether they should be "as obvious as possible" ([Dennis Ritchie's position](<https://groups.google.com/d/msg/comp.unix.wizards/rkPBbdTELl0/kcEeJBvE5VwJ>)) or whether they should be largely transparent. This particularly related to how "`..`" should be handled when the kernel is following a symlink. David Korn - author of the Korn Shell — made a fairly [concrete proposal](<https://groups.google.com/d/msg/comp.unix.wizards/rkPBbdTELl0/_Qxz3Va8R1EJ>) for the kernel to track which path was used to reach the "current working directory" so that "`..`" could lead back along that path. This never made it into any released Unix kernel, but does explain the behavior of the `pwd` built-in to `ksh` and related shells such as `bash`. 

Other concerns were raised over [what permission bits should mean](<https://groups.google.com/d/msg/comp.unix.wizards/rkPBbdTELl0/B1UCHhB3wbYJ>) and [whether hard links to symlinks made sense](<https://groups.google.com/d/msg/comp.unix.wizards/e99xBqj9WZE/q-7WtHjHg5cJ>). Such discussions have long since died down and POSIX came to define a set of semantics which, if not ideal, are at least uniformly implemented and fairly well understood. The task for pathname lookup in Linux is not to debate the meaning or value of symbolic links, but only to implement those semantics, correctly handling the various corner cases. 

There are two changes of note that happened in the recent rewrite. First, the recursive function calls were removed. There is still a recursive element to the algorithm because the problem itself is recursive, but it is now implemented using iteration and an explicit stack. This allows the symlink stack to be allocated separately from the system stack and so reduces pressure on what is often a limited resource. One concrete benefit of this is that code in the "lustre" filesystem that places extra limits on symlink recursion due to stack space concerns [can be removed](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/drivers/staging/lustre/lustre/llite/symlink.c?id=44163f30059e9869451999e77109a37abba8c968>). 

Second, the new code allows symlinks to be followed while in RCU-walk mode, at least some of the time. Previously this was not possible, partly because there are some awkward cases and partly because no one had bothered to do the work. 

The effort needed to understand the particular needs of symlinks in order to address these issues has resulted in some significant cleaning up of the code and simplifying of interfaces. The cleanup should remove at least [one source of confusion](<http://comments.gmane.org/gmane.linux.file-systems.f2fs/2435>) that surfaced recently. 

There are several basic issues that we will examine to understand the handling of symbolic links: the symlink stack, together with cache lifetimes, will help us understand the overall recursive handling of symlinks and lead to the special care needed for the final component. Then a consideration of access-time updates and summary of the various flags controlling lookup will finish the story.

#### The symlink stack

There are only two sorts of filesystem objects that can usefully appear in a path prior to the final component: directories and symlinks. Handling directories is quite straightforward: the new directory simply becomes the starting point at which to interpret the next component on the path. Handling symbolic links requires a bit more work. 

Conceptually, symbolic links could be handled by editing the path. If a component name refers to a symbolic link, then that component is replaced by the body of the link and, if that body starts with a '`/`', then all preceding parts of the path are discarded. This is what the "`readlink -f`" command does, though it also edits out "`.`" and "`..`" components. 

Directly editing the path string is not really necessary when looking up a path, and discarding early components is pointless as they aren't looked at anyway. Keeping track of all remaining components is important, but they can of course be kept separately; there is no need to concatenate them. As one symlink may easily refer to another, which in turn can refer to a third, we may need to keep the remaining components of several paths, each to be processed when the preceding ones are completed. These path remnants are kept on a stack of limited size. 

There are two reasons for placing limits on how many symlinks can occur in a single path lookup. The most obvious is to avoid loops. If a symlink referred to itself either directly or through intermediaries, then following the symlink can never complete successfully — the error `ELOOP` must be returned. Loops can be detected without imposing limits, but limits are the simplest solution and, given the second reason for restriction, quite sufficient. 

The second reason was [outlined recently](<http://thread.gmane.org/gmane.linux.kernel/1934390/focus=1934550>) by Linus:

Because it's a latency and DoS issue too. We need to react well to true loops, but also to "very deep" non-loops. It's not about memory use, it's about users triggering unreasonable CPU resources. 

Linux imposes a limit on the length of any pathname: `PATH_MAX`, which is 4096. There are a number of reasons for this limit; not letting the kernel spend too much time on just one path is one of them. With symbolic links you can effectively generate much longer paths so some sort of limit is needed for the same reason. Linux imposes a limit of at most 40 symlinks in any one path lookup. It previously imposed a further limit of eight on the maximum depth of recursion, but that was raised to 40 when a separate stack was implemented, so there is now just the one limit. 

The `nameidata` structure that we met in an earlier article contains a small stack that can be used to store the remaining part of up to two symlinks. In many cases this will be sufficient. If it isn't, a separate stack [is allocated](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n542>) with room for 40 symlinks. Pathname lookup will never exceed that stack as, once the 40th symlink is detected, an error is returned. It might seem that the name remnants are all that needs to be stored on this stack, but we need a bit more. To see that, we need to move on to cache lifetimes.

#### Storage and lifetime of cached symlinks

Like other filesystem resources, such as inodes and directory entries, symlinks are cached by Linux to avoid repeated costly access to external storage. It is particularly important for RCU-walk to be able to find and temporarily hold onto these cached entries, so that it doesn't need to drop down into REF-walk. 

While each filesystem is free to make its own choice, symlinks are typically stored in one of two places. Short symlinks are often stored directly in the inode. When a filesystem allocates a `struct inode` it typically allocates extra space to store private data (a common [object-oriented design pattern](<https://lwn.net/Articles/446317/>) in the kernel). This will sometimes include space for a symlink. The other common location is in the page cache, which normally stores the content of files. The pathname in a symlink can be seen as the content of that symlink and can easily be stored in the page cache just like file content. 

When neither of these are suitable, the next most likely scenario is that the filesystem will allocate some temporary memory and copy or construct the symlink content into that memory whenever it is needed. 

When the symlink is stored in the inode, it has the same lifetime as the inode which, itself, is protected by RCU or by a counted reference on the dentry. This means that the mechanisms that pathname lookup uses to access the dcache and icache (inode cache) safely are quite sufficient for accessing some cached symlinks safely. In these cases, the `i_link` pointer in the inode is set to point to wherever the symlink is stored and it can be accessed directly whenever needed. 

When the symlink is stored in the page cache or elsewhere, the situation is not so straightforward. A reference on a dentry or even on an inode does not imply any reference on cached pages of that inode, and even an `rcu_read_lock()` is not sufficient to ensure that a page will not disappear. So, for these symlinks, the pathname lookup code needs to ask the filesystem to provide a stable reference and, significantly, needs to release that reference when it is finished with it. 

Taking a reference to a cache page is often possible even in RCU-walk mode. It does require making changes to memory, which is best avoided, but that isn't necessarily a big cost and it is better than dropping out of RCU-walk mode completely. Even filesystems that allocate space to copy the symlink into can use `GFP_ATOMIC` to often successfully allocate memory without the need to drop out of RCU-walk. If a filesystem cannot successfully get a reference in RCU-walk mode, it must return `-ECHILD` and `unlazy_walk()` will be called to return to REF-walk mode in which the filesystem is allowed to sleep. 

The place for all this to happen is the `i_op->follow_link()` inode method. In the present mainline code this is never actually called in RCU-walk mode as the rewrite is not quite complete. It is likely that in a future release this method will be passed an `inode` pointer when called in RCU-walk mode so it both (1) knows to be careful, and (2) has the validated pointer. Much like the `i_op->permission()` method we looked at previously, `->follow_link()` would need to be careful that all the data structures it references are safe to be accessed while holding no counted reference, only the RCU lock. Though getting a reference with `->follow_link()` is not yet done in RCU-walk mode, the code is ready to release the reference when that does happen. 

This need to drop the reference to a symlink adds significant complexity. It requires a reference to the inode so that the `i_op->put_link()` inode operation can be called. In REF-walk, that reference is kept implicitly through a reference to the dentry, so keeping the `struct path` of the symlink is easiest. For RCU-walk, the pointer to the inode is kept separately. To allow switching from RCU-walk back to REF-walk in the middle of processing nested symlinks we also need the `seq` number for the dentry so we can confirm that switching back was safe. 

Finally, when providing a reference to a symlink, the filesystem also provides an opaque "cookie" that must be passed to `->put_link()` so that it knows what to free. This might be the allocated memory area, or a pointer to the `struct page` in the page cache, or something else completely. Only the filesystem knows what it is. 

In order for the reference to each symlink to be dropped when the walk completes, whether in RCU-walk or REF-walk, the [symlink stack](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n506>) needs to contain, along with the path remnants:

  * the `struct path` to provide a reference to the inode in REF-walk
  * the `struct inode *` to provide a reference to the inode in RCU-walk
  * the `seq` to allow the path to be safely switched from RCU-walk to REF-walk
  * the `cookie` that tells `->put_path()` what to put.

This means that each entry in the symlink stack needs to hold five pointers and an integer instead of just one pointer (the path remnant). On a 64-bit system, this is about 40 bytes per entry; with 40 entries it adds up to 1600 bytes total, which is less than half a page. So it might seem like a lot, but is by no means excessive. 

Note that, in a given stack frame, the path remnant (`name`) is not part of the symlink that the other fields refer to. It is the remnant to be followed once that symlink has been fully parsed. 

#### Following the symlink

The main loop in [`link_path_walk()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1859>) iterates seamlessly over all components in the path and all of the non-final symlinks. As symlinks are processed, the `name` pointer is adjusted to point to a new symlink, or is restored from the stack, so that much of the loop doesn't need to notice. Getting this `name` variable on and off the stack is very straightforward; pushing and popping the references is a little more complex. 

When a symlink is found, [`walk_component()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1695>) returns the value `1` (`0` is returned for any other sort of success, and a negative number is, as usual, an error indicator). This causes [`get_link()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n966>) to be called; it then gets the link from the filesystem. Providing that operation is successful, the old path `name` is placed on the stack, and the new value is used as the `name` for a while. When the end of the path is found (i.e. `*name` is `'\0'`), the old `name` is restored off the stack and path walking continues. 

Pushing and popping the reference pointers (inode, cookie, etc.) is more complex in part because of the desire to handle tail recursion. When the last component of a symlink itself points to a symlink, we want to pop the symlink-just-completed off the stack before pushing the symlink-just-found to avoid leaving empty path remnants that would just get in the way. 

It is most convenient to push the new symlink references onto the stack in `walk_component()` immediately when the symlink is found; `walk_component()` is also the last piece of code that needs to look at the old symlink as it walks that last component. So it is quite convenient for `walk_component()` to release the old symlink and pop the references just before pushing the reference information for the new symlink. It is guided in this by two flags; `WALK_GET`, which gives it permission to follow a symlink if it finds one, and `WALK_PUT`, which tells it to release the current symlink after it has been followed. `WALK_PUT` is tested first, leading to a call to [`put_link()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n840>). `WALK_GET` is tested subsequently (by [`should_follow_link()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1676>)) leading to a call to [`pick_link()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1642>), which sets up the stack frame. 

#### Symlinks with no final component

A pair of special-case symlinks deserve a little further explanation. Both result in a new `struct path` (with mount and dentry) being set up in the `nameidata`, and result in `get_link()` returning `NULL`. 

The more obvious case is a symlink to "`/`". All symlinks starting with "`/`" are detected in `get_link()`, which resets the `nameidata` to point to the effective filesystem root. If the symlink only contains "`/`" then there is nothing more to do, no components at all, so `NULL` is returned to indicate that the symlink can be released and the stack frame discarded. 

The other case involves things in `/proc` that look like symlinks but aren't really.

```
$ ls -l /proc/self/fd/1
        lrwx------ 1 neilb neilb 64 Jun 13 10:19 /proc/self/fd/1 -> /dev/pts/4
```

Every open file descriptor in any process is represented in `/proc` by something that looks like a symlink. It is really a reference to the target file, not just the name of it. When you `readlink()` these objects you get a name that might refer to the same file — unless it has been unlinked or mounted over. When `walk_component()` follows one of these, the `->follow_link()` method in "procfs" doesn't return a string name, but instead calls [`nd_jump_link()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n826>), which updates the `nameidata` in place to point to that target. `->follow_link()` then returns `NULL`. Again there is no final component and `get_link()` reports this by leaving the `last_type` field of `nameidata` as `LAST_BIND`. 

#### Following the symlink in the final component

All this leads to `link_path_walk()` walking down every component, and following all symbolic links it finds, until it reaches the final component. This is just returned in the `last` field of `nameidata`. For some callers, this is all they need; they want to create that `last` name if it doesn't exist or give an error if it does. Other callers will want to follow a symlink if one is found, and possibly apply special handling to the last component of that symlink, rather than just the last component of the original file name. These callers potentially need to call `link_path_walk()` again and again on successive symlinks until one is found that doesn't point to another symlink. 

This case is handled by the relevant caller of `link_path_walk()`, such as [`path_lookupat()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2088>), using a loop that calls `link_path_walk()`, and then handles the final component. If the final component is a symlink that needs to be followed, then [`trailing_symlink()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2062>) is called to set things up properly and the loop repeats, calling `link_path_walk()` again. This could loop as many as 40 times if the last component of each symlink is another symlink. 

The various functions that examine the final component and possibly report that it is a symlink are `lookup_last()`, `mountpoint_last()`, and `do_last()`, each of which use the same convention as `walk_component()` of returning `1` if a symlink was found that needs to be followed. Of these, `do_last()` is the most interesting as it is used for opening a file. Part of `do_last()` runs with `i_mutex` held and this part is in a separate function: `lookup_open()`. 

Explaining [`do_last()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2984>) completely is beyond the scope of this article, but a few highlights should help those interested in exploring the code.

  1. Rather than just finding the target file, `do_last()` needs to open it. If the file was found in the dcache, then `vfs_open()` is used for this. If not, then `lookup_open()` will either call `atomic_open()` (if the filesystem provides it) to combine the final lookup with the open, or will perform the separate `lookup_real()` and `vfs_create()` steps directly. In the later case, the actual "open" of this newly found or created file will be performed by `vfs_open()`, just as if the name were found in the dcache.

  2. `vfs_open()` can fail with `-EOPENSTALE` if the cached information wasn't quite current enough. Rather than restarting the lookup from the top with `LOOKUP_REVAL` set, `lookup_open()` is called instead, giving the filesystem a chance to resolve small inconsistencies. If that doesn't work, only then is the lookup restarted from the top.

  3. An open with `O_CREAT` _does_ follow a symlink in the final component, unlike other creation system calls (like `mkdir`). So the sequence:

```
ln -s bar /tmp/foo
             echo hello > /tmp/foo
```

will create a file called `/tmp/bar`. This is not permitted if `O_EXCL` is set but otherwise is handled for an `O_CREAT` open much like for a non-creating open: `should_follow_link()` returns `1`, and so does `do_last()`, so that `trailing_symlink()` gets called and the open process continues on the symlink that was found.

#### Updating the access time

We previously said of RCU-walk that it would "take no locks, increment no counts, leave no footprints." We have since seen that some "footprints" can be needed when handling symlinks as a counted reference (or even a memory allocation) may be needed. But these footprints are best kept to a minimum. 

One other place where walking down a symlink can involve leaving footprints in a way that doesn't affect directories is in updating access times. In Unix (and Linux) every filesystem object has a "last accessed time", or "atime". Passing through a directory to access a file within is not considered to be an access for the purposes of atime; only listing the contents of a directory can update its atime. Symlinks are different it seems. Both reading a symlink (with `readlink()`) and looking up a symlink on the way to some other destination can update the atime on that symlink. 

It is not clear why this is the case; POSIX has little to say on the subject. The [clearest statement](<http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap04.html#tag_04_08>) is that, if a particular implementation updates a timestamp in a place not specified by POSIX, this must be documented ""except that any changes caused by pathname resolution need not be documented"". This seems to imply that POSIX doesn't really care about access-time updates during pathname lookup. 

An examination of history shows that, prior to [Linux 1.3.87](<https://git.kernel.org/cgit/linux/kernel/git/history/history.git/diff/fs/ext2/symlink.c?id=f806c6db77b8eaa6e00dcfb6b567706feae8dbb8>), the ext2 filesystem, at least, didn't update atime when following a link. Unfortunately we have no record of why that behavior was changed. 

In any case, access time must now be updated and that operation can be quite complex. Trying to stay in RCU-walk while doing it is best avoided. Fortunately it is often permitted to skip the atime update. Because atime updates cause performance problems in various areas, Linux supports the `relatime` mount option, which generally limits the updates of atime to once per day on files that aren't being changed (and symlinks never change once created). Even without `relatime`, many filesystems record atime with a one-second granularity, so only one update per second is required. 

It is easy to test if an atime update is needed while in RCU-walk mode and, if it isn't, the update can be skipped and RCU-walk mode continues. Only when an atime update is actually required does the path walk drop down to REF-walk. All of this is handled in the `get_link()` function.

#### A few flags

A suitable way to wrap up this tour of pathname walking is to list the various flags that can be stored in the `nameidata` to guide the lookup process. Many of these are only meaningful on the final component, others reflect the current state of the pathname lookup. And then there is `LOOKUP_EMPTY`, which doesn't fit conceptually with the others. If this is not set, an empty pathname causes an error very early on. If it is set, empty pathnames are not considered to be an error. 

#### Global state flags

We have already met two global state flags: `LOOKUP_RCU` and `LOOKUP_REVAL`. These select between one of three overall approaches to lookup: RCU-walk, REF-walk, and REF-walk with forced revalidation. 

`LOOKUP_PARENT` indicates that the final component hasn't been reached yet. This is primarily used to tell the audit subsystem the full context of a particular access being audited. 

`LOOKUP_ROOT` indicates that the `root` field in the `nameidata` was provided by the caller, so it shouldn't be released when it is no longer needed. 

`LOOKUP_JUMPED` means that the current dentry was chosen not because it had the right name but for some other reason. This happens when following "`..`", following a symlink to "`/`", crossing a mount point or accessing a "`/proc/$PID/fd/$FD`" symlink. In this case the filesystem has not been asked to revalidate the name (with `d_revalidate()`). In such cases the inode may still need to be revalidated, so `d_op->d_weak_revalidate()` is called if `LOOKUP_JUMPED` is set when the look completes — which may be at the final component or, when creating, unlinking, or renaming, at the penultimate component.

#### Final-component flags

Some of these flags are only set when the final component is being considered. Others are only checked for when considering that final component. 

`LOOKUP_AUTOMOUNT` ensures that, if the final component is an automount point, then the mount is triggered. Some operations would trigger it anyway, but operations like `stat()` deliberately don't. `statfs()` needs to trigger the mount but otherwise behaves a lot like `stat()`, so it sets `LOOKUP_AUTOMOUNT`, as does `quotactl()` and the handling of "`mount --bind`". 

`LOOKUP_FOLLOW` has a similar function to `LOOKUP_AUTOMOUNT` but for symlinks. Some system calls set or clear it implicitly, while others have API flags such as `AT_SYMLINK_FOLLOW` and `UMOUNT_NOFOLLOW` to control it. Its effect is similar to `WALK_GET` that we already met, but it is used in a different way. 

`LOOKUP_DIRECTORY` insists that the final component is a directory. Various callers set this and it is also set when the final component is found to be followed by a slash. 

Finally `LOOKUP_OPEN`, `LOOKUP_CREATE`, `LOOKUP_EXCL`, and `LOOKUP_RENAME_TARGET` are not used directly by the VFS but are made available to the filesystem and particularly the `->d_revalidate()` method. A filesystem can choose not to bother revalidating too hard if it knows that it will be asked to open or create the file soon. These flags were previously useful for `->lookup()` too but with the introduction of `->atomic_open()` they are less relevant there.

#### End of the road

Despite its complexity, all this pathname lookup code appears to be in good shape — various parts are certainly easier to understand now than even a couple of releases ago. But that doesn't mean it is "finished". As already mentioned, RCU-walk currently only follows symlinks that are stored in the inode so, while it handles many ext4 symlinks, it doesn't help with NFS, XFS, or Btrfs. That support is not likely to be long delayed. 

There is also room for new enhancements. Having a single mutex to serialize all changes and uncached lookups in a directory can cause problems in some scenarios. As Linus said while [discussing the issue](<http://thread.gmane.org/gmane.linux.file-systems/96389/focus=96403>): ""anyway, just grepping for 'i_mutex' made me almost cry."" There is no immediate solution apparent, but it is likely that something could be done if sufficient motivation were found. 

A much simpler change that has been suggested is to add new lookup flags for ["no symlinks" and "no dotdot"](<http://thread.gmane.org/gmane.linux.file-systems/95648/focus=96300>). This could be possibly used by Samba, or by the Apache web server to handle lookup more efficiently when the "`FollowSymlinks`" directive is not in effect. This would need little more than an agreement on the correct API — so maybe not so easy after all. 

But these are all issues for the future. For now it is good to have something that works, that handles all the corner cases, that is really very efficient, and that is even documented.
