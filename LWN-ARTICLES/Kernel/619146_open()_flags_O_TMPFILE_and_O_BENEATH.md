---
title: "open() flags: O_TMPFILE and O_BENEATH"
url: https://lwn.net/Articles/619146/
date: "November 5, 2014"
category: O BENEATH; O TMPFILE
author: "By Jonathan Corbet November 5, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 5, 2014

Not much happens on a Linux system without one or more calls to `open()` or one of its variants. There is no other way to create a file or access a file that already exists. It thus follows that the various flags that control the behavior of `open()` have a significant effect on the functionality of the system as a whole. Here, we'll look at two specific `open()` flags; one of them is a relatively recent addition to the kernel, while the other is still in the proposal stage. 

#### O_TMPFILE

The `O_TMPFILE` flag has been discussed a few times in these pages; the [abrupt nature of its addition](<https://lwn.net/Articles/562294/>) meant that it had little review and a fair number of post-merge problems. The concept behind this flag is simple enough: it requests the creation of a file with no associated directory entry. It is thus meant for temporary files that will not be opened by any other process. 

Eric Rannaud recently [asked a question](<https://lwn.net/Articles/619147/>): what should happen when a process makes a call like the following? 

```
int fd = open("/tmp", O_TMPFILE | O_RDWR, 0);
```

The flags request the creation of a writable temporary file, but the third argument (the file mode) says that there should be no access (read or write) allowed. As it happens, POSIX is clear enough about this situation when a file is created with ordinary `O_CREAT`: the provided mode only applies _after_ the creation of the file. So, while a process can create a file that it cannot itself access in general, it can still get a working file descriptor in the act of creation itself. 

As it happens, though, file creation with `O_TMPFILE` does not work that way; the file mode is applied from the beginning, so the `open()` call listed above will fail. This behavior was widely recognized to be a bug, and Eric's fix was merged for the 3.18-rc3 release. But there are a couple of interesting side notes that are worth looking at. 

One is that this call: 

```
int fd = open("/tmp", O_TMPFILE | O_RDONLY, 0666);
```

will still fail. When the `O_TMPFILE` feature was implemented, it seemed that there was no use case for a temporary file that could not be written to, so this case (`O_TMPFILE` with `O_RDONLY`) was explicitly forbidden. But it [turns out](<https://lwn.net/Articles/619148/>) that there is a use case for this type of file: creating an empty file with a specific set of extended attributes atomically. The `open()` call would be followed by one or more `fsetxattr()` calls; once everything is in place, `linkat()` can be used to make the file visible in the filesystem. Linus initially [agreed](<https://lwn.net/Articles/619149/>) that this use case should be supported, but later [changed his mind](<https://lwn.net/Articles/619224/>). So read-only `O_TMPFILE` files will remain unsupported. 

Amusingly, the original bug was discovered while digging into [a related glibc bug](<https://sourceware.org/bugzilla/show_bug.cgi?id=17523>). It seems that, when `O_TMPFILE` is used, the mode argument isn't passed into the kernel at all. In the case of `open()` on x86-64 machines, things work out of sheer luck: the mode argument just happens to be sitting in the right register when glibc makes the call into the kernel. Things do not work as well with `openat()`, though, with the result that, in current glibc installations, `O_TMPFILE` cannot be used with `openat()` at all. The bug is well understood and should be fixed soon. 

#### O_BENEATH

When a developer makes a call to `openat()`, they will normally expect that the file being opened or created will be located in the specified directory. As is often the case, though, surprises lurk for the unwary. Trouble can come from a surprising symbolic link or deliberately malicious input; either way, it can lead to files being created or opened where they should not be. 

David Drysdale has a solution in the form of [the `O_BENEATH` flag](<https://lwn.net/Articles/619151/>) for `openat()`. If this flag is included in the call, the file being accessed must exist in or below the directory provided. The enforcement of this rule is simple enough: the provided path is constrained to not start with "`/`" or contain "`../`". Any symbolic links traversed while resolving the path must meet the same conditions. 

This feature was implemented as part of the filesystem access restrictions found in the [Capsicum](<https://lwn.net/Articles/604015/>) patch set. It turns out that there are other potential users as well, though. In particular, when combined with a secure computing ("seccomp") filter, `O_BENEATH` can be used to safely give a sandboxed process a directory to create files in. 

The initial review concerns raised against this patch have been addressed in the current version. It is a relatively simple and non-invasive patch, so there is a reasonable chance that we'll see it enter the mainline during a near-future merge window.
