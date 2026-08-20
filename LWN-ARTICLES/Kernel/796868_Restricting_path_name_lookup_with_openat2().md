---
title: Restricting path name lookup with openat2()
url: https://lwn.net/Articles/796868/
date: "August 22, 2019"
category: "Filesystems-Virtual filesystem layer; System calls-openat2"
author: "By Jonathan Corbet August 22, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 22, 2019

Looking up a file given a path name seems like a straightforward task, but it turns out to be one of the more complex things the kernel does. Things get more complicated if one is trying to write robust (user-space) code that can do the right thing with paths that are controlled by a potentially hostile user. Attempts to make the [`open()` and `openat()` system calls](<http://man7.org/linux/man-pages/man2/open.2.html>) safer date back at least to an attempt to add [`O_BENEATH` in 2014](<https://lwn.net/Articles/619146/>), but numerous problems remain. Aleksa Sarai, who has [been working in this area](<https://lwn.net/Articles/767547/>) for a while, has now concluded that a new version of `openat()`, naturally called [`openat2()`](<https://lwn.net/ml/linux-kernel/20190820033406.29796-1-cyphar@cyphar.com/>), is required to truly solve this problem. 

The immediate purpose behind `openat2()` is to allow a program to safely open a path that is possibly under the control of an attacker; in practice, that means placing restrictions on how the lookup process will be carried out. Past attempts have centered around adding new flags to `openat()`, but there are a couple of problems with that approach: `openat()` doesn't check for unknown flags, and the number of available bits for new flags is not large. The failure to check for unknown flags is [a well-known antipattern](<https://lwn.net/Articles/588444/>). A program using a path-restricting flag needs to know whether the requested behavior is understood by the kernel or not; the alternative is to accept security vulnerabilities on kernels that do not implement those flags. 

Changing `openat()` to return errors when passed unknown flags is not an option; that would almost certainly break existing code. So the only alternative is to create a new system call that _does_ check its flags; thus `openat2()`: 

```
struct open_how {
    	__u32 flags;
    	union {
    	    __u16 mode;
    	    __u16 upgrade_mask;
    	};
    	__u16 resolve;
    	__u64 reserved[7]; /* must be zeroed */
        };
    
        int openat2(int dfd, const char *filename, const struct open_how *how);
```

(Note that, as always, this is how the system call is presented at the user-space boundary; C libraries could expose something different.) 

Rather than add another argument for the new flags, Sarai opted to move all of the behavior-affecting information into a separate structure. Among other things, this means that, unlike `open()` and `openat()`, the new system call always has the same number of arguments. The `flags` field holds the same `O_` flags understood by `open()` and `openat()`, but unknown flags in this field will result in an error. Similarly, the `mode` field contains the permission bits used when a new file is created. 

The `resolve` field, instead, contains a new set of flags that control how pathname lookup will be performed. The flags implemented in the patch set are: 

> `RESOLVE_NO_XDEV`
>      The lookup process will not be allowed to cross any mount points (including bind mounts). In other words, the file to be opened must reside on the same mount as the `dfd` descriptor (or the current working directory if `dfd` is passed as `AT_FDCWD`). 
> 
> `RESOLVE_NO_MAGICLINKS`
>      There are relatively few types of objects that can be found in a filesystem directory; they include regular files, directories, devices, FIFOs, and symbolic links. With this option, Sarai is in essence acknowledging that there is another type that has been lurking in plain sight for years: the "magic link". Examples include the links found in `/proc/_PID_ /fd` directories; they are implemented by the kernel and have some possibly surprising properties. 
> 
> The presence of this flag will prevent a path lookup operation from traversing through one of these magic links, thus blocking (for example) attempts to escape from a container via a `/proc` entry for an open file descriptor. 
> 
> `RESOLVE_NO_SYMLINKS`
>      Blocks any traversal through symbolic links, including magic links. This option differs from the `O_NOFOLLOW` flag in that it prevents following a link at any point in the lookup process, while `O_NOFOLLOW` only applies to the last component in the path. 
> 
> `RESOLVE_BENEATH`
>      The lookup process is contained within the directory tree below the starting point; attempts to use components like "`../`" to escape that tree will generate an error. 
> 
> `RESOLVE_IN_ROOT`
>      This flag causes the lookup process to behave as if a [`chroot()`](<http://man7.org/linux/man-pages/man2/chroot.2.html>) to the starting point had been performed. Absolute paths will begin relative to the starting directory, and "`../`" will not proceed above that directory. Some work has been done to make `RESOLVE_IN_ROOT` free of some of the race conditions that plague `chroot()`; see [this changelog](<https://lwn.net/ml/linux-kernel/20190820033406.29796-7-cyphar@cyphar.com/>) for some details. 

The issue of magic links appears a few times in this patch set. The `RESOLVE_NO_MAGICLINKS` option prevents them from being traversed (or opened), but it turns out that there are numerous cases where it is indeed useful to open such links. The problem is that allowing that to happen can be dangerous; the [`runc` container breakout vulnerability](<https://lwn.net/Articles/781013/>) reported in February was the result of hostile code using the `/proc/_PID_ /exe` link to open the `runc` binary for write access. That can make for leaky containers, to put it mildly. 

The [first patch](<https://lwn.net/ml/linux-kernel/20190820033406.29796-2-cyphar@cyphar.com/>) in the series changes the semantics of `open()` (in all variants) when magic links are involved: while previously the permission bits on the magic link itself were ignored, now they are taken into account. Then, for example, the permissions for `/proc/_PID_ /exe` are [changed in the kernel](<https://lwn.net/ml/linux-kernel/20190820033406.29796-3-cyphar@cyphar.com/>) to disallow opening for write access, blocking the `runc` breakout attack. 

This change enables one other feature provided by `openat2()` (and, indeed, `openat()` as well). There is a new flag (`O_EMPTYPATH`) that causes the path argument to be ignored; instead, the call will simply reopen the file descriptor passed as the `dfd` argument using the new mode provided. A common use case is to reopen a file descriptor initially opened with `O_PATH` to gain access to file contents or metadata — access which is otherwise not possible with an `O_PATH` descriptor (see [the man page](<http://man7.org/linux/man-pages/man2/open.2.html>) for details on `O_PATH`). Programs have typically done this sort of reopening using a path into `/proc/_PID_ /fd`, but `O_EMPTYPATH` will work even if `/proc` is not available. 

Finally, the new API also allows the placement of limits on how a file descriptor created with `O_PATH` can be "upgraded" as described above. When `openat2()` is used to open an `O_PATH` file descriptor, the `upgrade_mask` field in the `how` structure can be used to limit the access that can be obtained by reopening in the future. Specifying `UPGRADE_NOREAD` will prevent reopening with read access, and `UPGRADE_NOWRITE` will prevent the acquisition of write access. This restriction can limit the damage should a hostile program obtain access to an `O_PATH` file descriptor. 

Previous versions of this patch set have generated a fair amount of discussion. The relative quiet after this posting may reflect the fact that most of the concerns raised have been addressed over time — or possibly just that the people who would comment on it are attending the Linux Security Summit and falling behind on email. Either way, there is a clear demand for the ability to restrict how file path names are traversed so, sooner or later, some version of this patch set seems likely to find its way into the mainline.
