---
title: Six (or seven) new system calls for filesystem mounting
url: https://lwn.net/Articles/759499/
date: "July 12, 2018"
category: "Filesystems-Mounting"
author: "By Jonathan Corbet July 12, 2018"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 12, 2018

Mounting filesystems is a complicated business. The kernel supports a wide variety of filesystem types, and each has its own, often extensive set of options. As a result, the [`mount()` system call](<http://man7.org/linux/man-pages/man2/mount.2.html>) is complex, and the [list of mount options](<http://man7.org/linux/man-pages/man8/mount.8.html#FILESYSTEM-INDEPENDENT_MOUNT_OPTIONS>) is a rather long read. But even with all of that complexity, `mount()` does not do everything that users would like. For example, the options for a mount operation must all fit within a single 4096-byte page — the fact that this is a problem for some users is illustrative in its own right. The problems with `mount()` have come up at various meetings, including [at the 2018 Linux Storage, Filesystem, and Memory-Management Summit](<https://lwn.net/Articles/753473/>). A [set of patches](<https://lwn.net/ml/linux-kernel/153126248868.14533.9751473662727327569.stgit@warthog.procyon.org.uk/>) implementing a new approach is getting closer to being ready, but it features some complexity of its own and there are some remaining concerns about the proposed system-call API. 

This patch set, from David Howells, is in its ninth revision. It makes extensive changes within the virtual filesystem layer to create the concept of a "filesystem context" that describes a specific mount operation. The questions about the internal changes have mostly been resolved at this point; things seem about ready to go in at that level. But the patch set also replaces the `mount()` system call with a rather more complex set of operations. (To be precise, `mount()` would not go away as long as it is needed, but it is unlikely to gain new functionality after the new system calls go in.) 

#### The new way of mounting

In current kernels, a single `mount()` call does everything required to mount a filesystem at a specific location in the system hierarchy. With these patches applied, instead, the process would begin with a call to the new `fsopen()` system call: 

```
int fsopen(const char *fsname, unsigned int flags);
```

The `fsname` parameter identifies the type of the filesystem to be mounted — `ext4` or `nfs`, for example — while `flags` is either zero or `FSOPEN_CLOEXEC`. This call doesn't mount any filesystems, it just creates the context in which the mount operation can be described and carried out. The return value is a file descriptor representing that context. 

The next step is to provide the details for the mount to be performed; this is done by writing a series of strings to that file descriptor. The first character of the string is either "`s`" (to specify the source filesystem), "`o`" (to provide a mount option), or "`x`" (to execute a command). So a reasonable series of writes could be: 

```
s /dev/sda1
        o noatime
        x create
```

Note that these strings are not terminated by newlines; each `write()` call is supposed to convey exactly one of these strings. In this case, the strings written say that the filesystem found on `/dev/sda1` should be mounted with the `noatime` option. The final line (with the `create` command) brings the filesystem context into fully formed existence, but does not actually mount it anywhere. There is also a `reconfigure` command that can be used to change the settings in an existing context. 

Things can go wrong at any step, in which case the `write()` call will return an error. More detailed information about the problem can be had by reading from the file descriptor. This feature addresses one of the other problems with `mount()`: the inability to communicate the details of a problem to user space. 

Assuming all goes well, the next step is to mount the filesystem with a call to: 

```
int fsmount(int fd, unsigned int flags, unsigned int ms_flags);
```

The filesystem-context file descriptor created by `fsopen()` is passed as `fd` to `fsmount()`. Once again, the only flag for `flags` is `FSMOUNT_CLOEXEC`, while `ms_flags` describe how the mount is to be performed. They can be used to create an unbindable or slave mount, for example (see [this article](<https://lwn.net/Articles/689856/>) for details on mount types). Some of those flags, though, duplicate options like `noatime` or read-only. 

`fsmount()` returns another file descriptor corresponding to the newly mounted filesystem. Do note, though, that while the filesystem is "mounted", it has not been mounted at any specific location in the filesystem tree, so it will not be visible to users. Actually placing the filesystem into a mount namespace requires yet another system call: 

```
int move_mount(int from_dfd, const char *from_path,
                       int to_dfd, const char *to_path, unsigned int flags);
```

To put a mounted filesystem into a spot in the hierarchy, `move_mount()` would be called with the file descriptor from `fsmount()` passed as `from_dfd` (`from_path` would be `NULL`). The location where the filesystem should be placed is described by `to_dfd` and `to_path` in the usual manner for *`at()` system calls. Among other things, the `to_dfd` file descriptor will identify the mount namespace in which the mount appears — something that can be tricky to do currently. The `flags` argument is used to control behavior like following symbolic links or whether to automount filesystems when determining the source and destination locations. 

As might be expected, `move_mount()` can also be used to relocate a fully mounted filesystem within the tree. 

#### Other operations

That is the basic sequence of operations to mount a filesystem in the new order. But, of course, the real world is more complex than that. Users want to query filesystems, remount them into different namespaces, remount them with different options, and more. Three more system calls have been provided to make these actions possible; the first of those is `fsinfo()`: 

```
int fsinfo(int dfd, const char *filename,
    	       const struct fsinfo_params *params,
    	       void *buffer, size_t buf_size);
```

This call can be used to query just about any attribute of a mounted filesystem. It is somewhat complex; interested readers can see [the patch changelog](<https://lwn.net/ml/linux-kernel/153126269451.14533.13592791373864325188.stgit@warthog.procyon.org.uk/>) for details, or [the man page patch](<https://lwn.net/ml/linux-kernel/15519.1531263314@warthog.procyon.org.uk/>) for a _lot_ of details. 

If the goal is to create a new mount of an existing filesystem, a more straightforward path is to use `open_tree()`: 

```
int open_tree(unsigned int dfd, const char *pathname, unsigned int flags);
```

Without special flags, this call is similar to calling [`open()`](<http://man7.org/linux/man-pages/man2/open.2.html>) on a directory with the `O_PATH` flag set. It returns a file descriptor corresponding to that directory that can only be used for a small set of operations — `move_mount()`, for example. But with the `OPEN_TREE_CLONE` flag, it will make a copy of the filesystem mount that can then be mounted elsewhere; it can thus be used to create a bind mount. Add the `AT_RECURSIVE` flag, and a whole hierarchy can be cloned and made ready for mounting in a different context. 

Finally, there is `fspick()`: 

```
int fspick(unsigned int dirfd, const char *path, unsigned int flags);
```

This system call can be thought of as the equivalent of `fsopen()` for an existing mount point. It returns a file descriptor that can be written to in the same way to change the mount parameters; the "`x reconfigure`" string at the end creates the equivalent of a remount operation. 

#### Playing with fire

There is relatively little controversy around most of this work, perhaps because few people have the stamina to plow through a 32-part patch set deep in the virtual filesystem layer. The concerns that have been raised have to do with the configuration API for file descriptors returned by `fsopen()` and `fspick()`. Andy Lutomirski [was clear](<https://lwn.net/ml/linux-kernel/686E805C-81F3-43D0-A096-50C644C57EE3@amacapital.net/>) about his concerns, saying: ""I think you’re seriously playing with fire with the API"". His worry, [echoed](<https://lwn.net/ml/linux-kernel/CA+55aFzknfBLjg4-pRSCUeBFY3DWq04zX3PPyc949M0OygHktw@mail.gmail.com/>) by Linus Torvalds, is that the API based on `write()` calls could be dangerous. 

In particular, Lutomirski worried that an attacker might succeed in getting a setuid program to write to one of these file descriptors, giving that attacker access to files or devices that would otherwise be protected. This problem could be avoided by using the credentials of the process that created the file descriptor for all subsequent operations — something that is supposed to happen already — but that is not seen as a practical possibility; as Torvalds [noted](<https://lwn.net/ml/linux-kernel/CA+55aFznwKKy0Vj4Wxe16=2EUmgas_Dcr9QQ1tOJOQOEAVfr+w@mail.gmail.com/>), even code that tries to get that right often makes mistakes and ends up using the credentials of the process calling `write()` instead. 

Solving this problem requires changing the API so that a call to `write()` does not have arbitrary side effects in the kernel. One possibility is to create yet another system call and use it to communicate the mount parameters to the kernel; that would prevent problems resulting from a redirected write. The alternative, which seems likely to be the way things will go in the end, is to add a different system call to replace the "`x`" operation at the end of that series of writes. It would look something like: 

```
int fscommit(unsigned int fd, unsigned int cmd);
```

Where `fd` is the file descriptor for the under-construction mount point, and `cmd` is either `FSCOMMIT_CREATE` or `FSCOMMIT_RECONFIGURE`. The `CAP_SYS_ADMIN` capability would be required to perform this operation. The end result would be that, while an attacker might be able to convince a setuid program to write to the file descriptor, that attacker would not be able to actually make the changes effective without having already gained a high level of privilege. 

Regardless of the final conclusion, this patch set will need to go through at least one more round before it can be merged. Torvalds has also [complained](<https://lwn.net/ml/linux-kernel/CA+55aFzEjPUGZFk7PnM0T6YEn5uRrscgyCHyhc_cYz0m8ejdLA@mail.gmail.com/>) that the motivation behind this work is not well described: ""I sure want to see an explanation for *WHY* it adds 5000+ lines of core code"" (a lot of interesting information can be found in [Howells's response](<https://lwn.net/ml/linux-fsdevel/29128.1531356361@warthog.procyon.org.uk/>) to that request). There is clearly some work to be done still, so this work will probably not be ready for the next merge window. In the not-too-distant future, though, the `mount()` system call seems likely to become obsolete.
