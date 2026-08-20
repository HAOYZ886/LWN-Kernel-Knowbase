---
title: Restricting pathname resolution with AT_NO_JUMPS
url: https://lwn.net/Articles/723057/
date: "May 17, 2017"
category: "Filesystems-Virtual filesystem layer"
author: "May 17, 2017 This article was contributed by Nur Hussein"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

May 17, 2017

This article was contributed by Nur Hussein

On April 29, Al Viro [posted a patch](<https://lwn.net/Articles/721443/>) on the linux-api mailing list adding a new flag to be used in conjunction with the `...at()` family of system calls. The flag is for containing pathname resolution to the same filesystem and subtree as the given starting point. This is a useful feature to have for implementing file I/O in programs that accept pathnames as untrusted user input. The ensuing discussion made it clear that there were multiple use cases for such a feature, especially if the granularity of its restrictions could be increased. 

As an example use case, consider a web server that accepts requests for documents along with a relative path to said documents from the root of the server data directory. It is imperative that the user-supplied pathname sent to a web server is not allowed to name files outside of the web-server subtree. If the web server could use file I/O system calls that guaranteed that any given path will never break out of a certain subdirectory tree, it would give an additional layer of security to the server.

The `...at()` family of system calls (such as `[openat()](<http://man7.org/linux/man-pages/man2/open.2.html>)`, `[fstatat()](<http://man7.org/linux/man-pages/man3/fstatat.3p.html>)`, `[mknodat()](<http://man7.org/linux/man-pages/man2/mknod.2.html>)`, etc.) are a relatively new series of POSIX filesystem interfaces, similar to `open()`, `fstat()`, `mknod()`, and friends, but allowing the start point for relative pathnames to be something other than the current working directory. For example, the `openat()` system call's prototype is: 

```
int openat(int dirfd, const char *pathname, int flags);
```

A call to `openat()` will try to open the file specified by the `pathname`. If the pathname is a relative path, such as `../home/user1`, `openat()` will try and walk the directory structure relative to the directory bound to the file descriptor `dirfd`, instead of the current working directory (which is what `open()` does). The behavior of this and other calls in the same family can be further altered with the `flags` parameter.

Viro's proposed flag (initially called `AT_NO_JUMPS`) for the `..at()` calls would restrict the directory traversal of those calls to the same subtree as the starting point and, in any case, within the same filesystem. When the flag is set, the `...at()` call affected would fail with `-ELOOP` upon encountering an absolute pathname, an absolute symbolic link, a procfs-style symbolic link, a pathname that results in the traversal of a mount point, or any relative path that starts with "`..`". This flag thus confines pathname lookups to subdirectories of the starting point that are contained within the same mount point, while allowing the traversal of relative symbolic links that do not violate the aforementioned rules. The error code `-ELOOP` is used by Unix-like operating systems to tell the user that too many symbolic links were encountered during pathname lookup, but it is used here as a placeholder. The patch only implements `AT_NO_JUMPS` for `fstatat()` and friends, but the proposal is for it to be extended to all the `...at()` calls. 

Jann Horn [commented](<http://marc.info/?l=linux-kernel&m=149366024305125&w=2>) that this proposal is somewhat similar to the [`O_BENEATH` functionality](<https://lwn.net/Articles/619146/>) that was [sent to the Linux kernel list](<https://lwn.net/Articles/619151/>) by David Drysdale in November 2014, but was ultimately not merged. The `O_BENEATH` flag is a port of a feature from [Capsicum](<https://lwn.net/Articles/604015/ >), a project that seeks to provide the ability to use capabilities to let applications sandbox themselves with a high degree of control. Horn noted that, while the functionality is similar, the intended use case for `O_BENEATH` is application sandboxing, whereas the `AT_NO_JUMPS` flag is to enable user programs to limit their own filesystem access. Viro [commented](<http://marc.info/?l=linux-api&m=149394423423941&w=2>) that, unlike `O_BENEATH`, `AT_NO_JUMPS` does allow relative symbolic links, which he thinks is the saner option:

It's not quite O_BENEATH, and IMO it's saner that way - a/b/c/../d is bloody well allowed, and so are relative symlinks that do not lead out of the subtree. If somebody has a good argument in favour of flat-out ban on .. (_other_ than "other guys do it that way, and it doesn't need to make sense 'cuz security!!1!!!", please), I'd be glad to hear it. 

Andy Lutomirski [suggested](<http://marc.info/?l=linux-fsdevel&m=149351485126402&w=2>) splitting the flag into two, ""one to prevent moving between mounts and one for everything else"". This is because web servers and the like will probably be fine with mount-point traversal; they only need the directory containment feature. Horn [concurred](<http://marc.info/?l=linux-api&m=149366026005128&w=2>) with Lutomirski about the usefulness of the split.

Viro agreed that the no-mount-point-crossing policy from `AT_NO_JUMPS` could be split out into a separate flag. He proposed `AT_XDEV` for preventing mount point crossing, and the original flag be renamed to `AT_BENEATH` to match the functionality of `O_BENEATH`, which does allow crossing mount points. The returned error for crossing mount points when `AT_XDEV` is enabled could be the obvious `-EXDEV`, while the error for `AT_BENEATH` would still be `-ELOOP` (which Viro isn't too satisfied with, but nothing else has been suggested thus far). 

Linus Torvalds [liked the split](<http://marc.info/?l=linux-kernel&m=149394765324531&w=2>), but wanted to go even further:

So I would still like to split that NO_JUMP flag even more. I like the AT_BENEATH | AT_XDEV split, but I think XDEV should be split further, and I think the symlink avoidance should be split more too. 

As mentioned last time, at least for the git usage, even relative symlinks are a no-no - not because they'd escape, but simply because git wants to see the *unique* name, and resolve relative symlinks to either the symlink, or to the actual file it points to. So I think that we'd want an additional flag that says "no symlinks at all". And I think the "no mountpoint" traversal might be splittable too. 

Torvalds went on to say that, sometimes, the use case is just to guarantee that pathname resolution does not go above a certain point in the directory tree, regardless of whether the directory walk crosses mount points. However this should only be the case of non-bind mount points. Bind mount points are basically views of directories (or files) that are mounted in another place in the directory tree. Since bind mounts can be used to break the directory containment, they should not be allowed. However from the system's point of view, there is no difference between a mounted filesystem and a bind-mounted directory, and thus Torvalds is not sure if it a mount point can be tested if it is a bind mount or not. Viro [agreed](<http://marc.info/?l=linux-api&m=149395326325891&w=2>) that it wasn't testable, and thus cannot be handled as a special case.

Viro then proposed that the flags become `AT_BENEATH`, `AT_XDEV`, and `AT_NO_SYMLINKS`, which is the flag used when no symbolic links are allowed at all. This proposal raises a few questions on how to handle some of the combinations of these three flags. Viro asked what `AT_XDEV` should do with absolute symbolic links. Torvalds [replied](<http://marc.info/?l=linux-api&m=149395688726538&w=2>) that, while it might be more consistent to allow an absolute symbolic link to be traversed with `AT_XDEV` (but without `AT_BENEATH`) as long as the root directory is on the same mount point as the starting point, it would be more straightforward to just return `-EXDEV` on all absolute symbolic links. 

Next, if the apparently conflicting flags `AT_BENEATH` (which allows symbolic links) and `AT_NO_SYMLINKS` (which disallows all symbolic links) are invoked simultaneously, Viro suggested that `AT_NO_SYMLINKS` take precedence since it was convenient to implement, to which Torvalds agreed.

Finally, Viro asked what should happen when the final component of a pathname is a symbolic link when `AT_NO_SYMLINKS` is applied. Torvalds thinks it should be an error if the symbolic link is followed, except if paired with `AT_SYMLINK_NOFOLLOW`, which indicates that the user is fine with a "dangling symlink" at the end, which will not be followed.

If Viro's proposed changes are picked up in the mainline kernel, we should see more robust directory containment options in the Linux API, which application writers can in turn use for file I/O. Since the pathname traversal protection mechanism will be in the kernel, there will no longer be a need for each user program to do its own pathname checking. This should be a welcome feature for anyone writing applications that work with file and directory names as user input.
