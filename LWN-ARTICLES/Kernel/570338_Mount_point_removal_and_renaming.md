---
title: Mount point removal and renaming
url: https://lwn.net/Articles/570338/
date: "October 16, 2013"
category: "Filesystems-Mounting; Namespaces-User namespaces"
author: "By Jake Edge October 16, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
October 16, 2013

Mounting a filesystem is typically an operation restricted to the root user (or a process with `CAP_SYS_ADMIN`). There are ways to allow regular users to mount certain filesystems (e.g. removable devices like CDs or USB sticks), but that needs to be set up in advance by an administrator. In addition, [bind mounts](<http://docs.1h.com/Bind_mounts>), which mount a portion of an already-mounted filesystem in another location, always require privileges. [User namespaces](<https://lwn.net/Articles/532593/>) will allow any user to be root inside their own namespace—thus be able to mount files and filesystems in (currently) unexpected ways. As might be guessed, that can lead to some surprising behavior that a patch set from Eric W. Biederman is trying to address. 

The problem crops up when someone tries to delete or rename a file or directory that has been used as a mount point elsewhere. A user only needs read access to a file (and execute permissions to the directories in the path) to be able to use it as a mount point, which means that users can mount filesystems over files they don't own. When the owner of the file (or directory) goes to remove it, they get an `EBUSY` error—for no obvious reason. Biederman has [proposed](<https://lwn.net/Articles/569672/>) changing that with a set of patches that would allow the unlink or rename to proceed and to quietly unmount anything mounted there. 

For example, if two users were to set up new mount and user namespaces ("user1" creates "ns1", "user2" creates "ns2"), the existing kernel would give the following behavior: 

```
ns1$ ls foo
        f1   f2
        ns1$ mount foo /tmp/user2/bar
```

Over in the other namespace, user2 tries to remove their temporary directory: 

```
ns2$ ls /tmp/user2/bar
        ns2$ rmdir /tmp/user2/bar
        rmdir: failed to remove ‘bar’: Device or resource busy
```

The visibility of mounts in other mount namespaces is part of the problem. A user getting an `EBUSY` when they attempt to remove their own directory may not even be able to determine _why_ they are getting the error. They may not be able to see the mount on top of their file because it was made in another namespace. Coupled with user namespaces, this would allow unprivileged users to perform a denial of service attack against other users—including those more privileged. 

Biederman's patches first add mount tracking to the virtual filesystem (VFS) layer. That will allow the later patches to find any mounts associated with a particular mount point. Using that, all of the mounts for a given directory entry (dentry) can be unmounted, which is exactly what is done when a mount point is deleted or renamed. 

The idea was generally greeted favorably, but Linus Torvalds [raised an issue](<https://lwn.net/Articles/570476/>): some programs are written to expect that `rmdir()` on a non-empty directory has no side effects, as it just returns `ENOTEMPTY`. The existing behavior is to return `EBUSY` if the directory is a mount point, but under Biederman's patches, any mount on the directory would be unmounted before determining whether the directory is empty and can be removed. That essentially adds a side effect to `rmdir()` even if it fails. 

In addition, depending on the [mount propagation](<https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt>) settings, the mount in another namespace might be visible. So, a user looking at "their" directory may actually be seeing files that were mounted by another user. But if they try to delete the directory (or some program does), it might succeed because the underlying mount point directory is empty, which may violate the principle of least surprise. 

Torvalds was not at all sure that any application cares, but was concerned that it made the change to the semantics larger than needed. He also had a suggestion for a way forward: 

That said, I like the _concept_ of being able to remove a mount-point and the mount just goes away. But I do think that for sanity sake, it should have something like "if one of the mounts is in the current namespace, return -EBUSY". IOW, the patch-series would make the VFS layer _able_ to remove mount-points, but a normal rmdir() when something is mounted in that namespace would fail in order to give legacy behavior. 

Biederman agreed and proposed another [patch](<https://lwn.net/Articles/570478/>) that would cause `rmdir()` to fail with an `EBUSY` if there is a mount on the directory in the current mount namespace. Mounts in other mount namespaces would continue to be unmounted in that case. But there were some questions raised about whether renaming mount points (or `unlink()`ing file mount points) should get the same treatment. 

Serge E. Hallyn [asked](<https://lwn.net/Articles/570479/>): ""Do you think we should do the same thing for over-mounted file at vfs_unlink()?"" In other words: if the mount is atop a _file_ that is removed (`unlink()`ed), rather than a directory, should the same rule be applied? The question was eventually broadened to include `rename()` as well. At first, Biederman thought the rules should only apply to `rmdir()`, believing that the permissions in the enclosing directories should be sufficient to avoid any problems with the other two operations. But after some discussion with Miklos Szeredi and Andy Lutomirski, he [changed his mind](<https://lwn.net/Articles/570480/>). For consistency, as well as alleviating a race condition in earlier (pre-`UMOUNT_NOFOLLOW`) versions of the `fusermount` command, ""the most practical path I can see is to block unlink, rename, and rmdir if there is a mount in the local namespace"". 

The `fusermount` race comes about because of its attempt to ensure that the mount point it is unmounting does not change out from under it. A malicious user could replace the mount point with a symbolic link to some other filesystem, which the root-privileged `fusermount` would happily unmount. Earlier, Biederman had [seen](<https://lwn.net/Articles/570481/>) that problem as an insurmountable hurdle to his approach for fixing the `rmdir()` problem. But, not allowing mount point renames eliminates most of the concern with the `fusermount` race condition. There are still unlikely scenarios where an older `fusermount` binary and a newer kernel could be subverted to unmount any filesystem, but Szeredi, who is the FUSE maintainer, is [not overly worried](<https://lwn.net/Articles/570482/>). It should be noted that there are other ways to "win" that race even in existing kernels (by renaming a parent directory of the mount point, for example). 

New [patches](<https://lwn.net/Articles/570482/>) reflecting the changes suggested by various reviewers were posted on October 15\. Biederman is targeting the 3.13 kernel, so there is some more time for reviewers to weigh in. It is a change that interested folks should be paying attention to, as it does subtly change the longtime behavior of the kernel. 

It is, in some ways, another example of the unintended consequences of user namespaces. If user namespaces are not enabled, the problem is essentially just a source of potential confusion; it only becomes a denial of service when they are enabled. But, if distributions are to ever enable user namespaces, these kinds of problems need to be found and fixed.
