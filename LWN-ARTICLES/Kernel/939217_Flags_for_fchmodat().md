---
title: Flags for fchmodat()
url: https://lwn.net/Articles/939217/
date: "July 27, 2023"
category: "Releases-6.6; Symbolic links; System calls-fchmodat"
author: "By Jonathan Corbet July 27, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 27, 2023

The [`fchmodat()`](<https://man7.org/linux/man-pages/man3/chmod.3p.html>) system call on Linux hides a little secret: it does not actually implement all of the functionality that the man page claims (and that [POSIX calls for](<https://pubs.opengroup.org/onlinepubs/9699919799/functions/chmod.html>)). As a result, C libraries have to do a bit of a complicated workaround to provide the API that applications expect. That situation looks likely to change with the 6.6 kernel, though, as the result of [this patch series](<https://lwn.net/ml/linux-kernel/cover.1689092120.git.legion@kernel.org/>) posted by Alexey Gladkov. 

The prototype for `fchmodat()` is defined as: 

```
int fchmodat(int fd, const char *path, mode_t mode, int flag);
```

Its purpose is to change the permissions of the file identified by `path` to the given `mode`. In the style of all the `*at()` system calls, `fd` can be an open file descriptor referring to a directory; if `path` is relative, the lookup process will start at the directory indicated by `fd` rather than the current working directory. The `flag` argument can be either zero or `AT_SYMLINK_NOFOLLOW`. 

Support for `fchmodat()` was added to the Linux kernel for the 2.6.16 release in 2006 as part of a series from Ulrich Drepper adding a number of the `*at()` calls. That version of `fchmodat()`, though, did not include the `flag` argument, a situation that continues to the present. As a result, the kernel's `fchmodat()` implementation is not compliant with the specification, and is not what application developers will expect. That, in itself, is not entirely unusual; applications do not (usually) invoke system calls directly. Instead, they use wrappers in a low-level library, usually the C library, which do what is needed to provide the expected API. That is what happens here, but the result is not ideal. 

The POSIX specification defines the behavior of the `AT_SYMLINK_NOFOLLOW` flag as: ""If `path` names a symbolic link, then the mode of the symbolic link is changed"". That behavior differs from the default, where the mode of the file _pointed to_ by that link will be changed instead. There are two reasons why one might want a flag like this: to actually change permissions on a symbolic link, and, more importantly, to _prevent_ the changing of permissions on a real file by way of a symbolic link. Attackers have been known to use symbolic links to confuse a privileged program into changing file modes that should not be changed; using this flag will prevent such an outcome. 

If one looks at the (functionally identical) `fchmodat()` implementations in [the GNU C library](<https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/unix/sysv/linux/fchmodat.c;h=17eca54051ee28ba1ec3f9aed170a62630959143;hb=a492b1e5ef7ab50c6fdd4e4e9879ea5569ab0a6c#l28>) and [musl libc](<https://git.musl-libc.org/cgit/musl/tree/src/stat/fchmodat.c?id=718f363bc2067b6487900eddc9180c84e7739f80#n28>), two things jump out: implementing `AT_SYMLINK_NOFOLLOW` in user space is inelegant at best and, due to limitations in Linux itself, neither library is able to implement exactly what the specification says (but they are able to provide the important part). 

The C-library implementations start by opening the file indicated by the `fd` and `path` arguments to `fchmodat()` as an `O_PATH` file descriptor. Such a descriptor allows metadata operations, but cannot be used to read or write the file; thus, it does not require read or write permission on the file to open. That `open()` call also uses the `O_NOFOLLOW` flag; if the path ends with a symbolic link, that will cause the link itself to be opened, rather than the file pointed to. 

At this point, the C libraries do an `fstatat64()` call to determine what kind of file has just been opened; if the new file descriptor turns out to be a symbolic link, an `EOPNOTSUPP` failure status will be returned to the caller. The Linux kernel does not support changing the permission bits on a symbolic link in general (those bits have no real meaning anyway), so neither C-library implementation even tries. 

If the target is _not_ a symbolic link, the library could just issue a normal `fchmodat()` call with the given parameters and no flag. That, however, could open the door to a time-of-check-to-time-of-use vulnerability, where an attacker would replace the file with a symbolic link between the check and the mode change. So, instead, the library must change the mode bits on the file that it actually opened in the first step, without using the path name again. Unfortunately, the obvious way (using [`fchmod()`](<https://man7.org/linux/man-pages/man3/fchmod.3p.html>)) won't work, because that system call cannot operate on `O_PATH` file descriptors in many filesystems. So, instead, the C library generates the path for the open file descriptor under `/proc/self/fd`, then passes that to [`chmod()`](<https://man7.org/linux/man-pages/man3/chmod.3p.html>) to effect the mode change. 

This sequence seems unlikely to be the most efficient way to prevent the following of a symbolic link for an `fchmodat()` call. It also will fail to work in settings where `/proc` is not available. A much nicer solution would be to just implement the `AT_SYMLINK_NOFOLLOW` flag in the kernel, which already has the needed machinery to do so in an atomic and efficient manner. 

That is what Gladkov's patch series does: it creates a new `fchmodat2()` system call that implements the `AT_SYMLINK_NOFOLLOW` flag. Once this system call is available in released kernels, the C-library implementations can use it for their implementation of `fchmodat()`, bypassing the current workarounds. The result should be a faster and more robust implementation. Chances are that change will happen soon; VFS maintainer Christian Brauner has [applied the series](<https://lwn.net/ml/linux-kernel/20230711-befreien-unwiderruflich-c2265c61e514@brauner/>) and routed it into linux-next, meaning that it should be pushed during the 6.6 merge window. 

Interestingly, this is not the first attempt to add an `fchmodat2()` implementation; there were patches posted by [Rich Felker](<https://lwn.net/ml/linux-kernel/20200916002157.GO3265@brightrain.aerifal.cx/>) in 2020 and [Greg Kurz](<https://lore.kernel.org/all/148830142269.7103.7429913851447595016.stgit@bahia/>) in 2017. It is not entirely clear why the patches were not accepted at that time; it may be simply because VFS patches have occasionally tended to fall through the cracks over the years. The previous failure may be part of why Felker [responded rather negatively](<https://lwn.net/ml/linux-kernel/20230725184443.GA20050@brightrain.aerifal.cx/>) to [a suggestion](<https://lwn.net/ml/linux-kernel/104971.1690300714@warthog.procyon.org.uk/>) from David Howells that, perhaps, it would be better to add a new `set_file_attrs()` system call, with a number of new features, rather than completing `fchmodat()`. That suggestion has not gained much support, so Gladkov's attempt appears to be the one that will actually succeed; after 17 years in the kernel, `fchmodat()` should finally get in-kernel `AT_SYMLINK_NOFOLLOW` support.
