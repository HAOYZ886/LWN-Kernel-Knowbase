---
title: revoke() returns
url: https://lwn.net/Articles/262528/
date: "December 18, 2007"
category: revoke
author: "By Jonathan Corbet December 18, 2007"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 18, 2007

LWN last looked at Pekka Enberg's `revoke()` patch [in July, 2006](<http://lwn.net/Articles/192632/>). The purpose of this proposed system call is to completely disconnect all processes from a specific file, thus allowing a new process to have exclusive access to that file. There are a number of applications for this functionality, such as ensuring that a newly logged-in user is the only one able to access resources associated with the console - the sound device, for example. There are kernel developers who occasionally mutter ominously about unfixable security problems resulting from the lack of the ability to revoke open file descriptors - though they tend, for some reason, to not want to publish the details of those vulnerabilities. Any sort of real [malware scanning](<http://lwn.net/Articles/260918/>) application will also need to be able to revoke access to files determined to contain Bad Stuff. 

Pekka has recently posted [a new version of the patch](<http://lwn.net/Articles/262377/>), so a new look seems warranted. The first thing one notes is that the `revoke()` system call is gone; instead, the new form of the system call is: 

```
int revokeat(int dir_fd, const char *filename);
```

This call thus follows the form of a number of other, relatively new `*at()` system calls. Here, `filename` is the name of the file for which access is to be revoked; if it is an absolute pathname then `dir_fd` is ignored. Otherwise, `dir_fd` is an open file descriptor for the directory to be used as the starting point in the lookup of `filename`. The special value `AT_FDCWD` indicates the current working directory for the calling process. If the `revokeat()` call completes successfully, only file descriptors for `filename` which are created after the call will be valid. 

There is a new `file_operations` member created by this patch set: 

```
int (*revoke)(struct file *filp);
```

This function's job is to ensure that any outstanding I/O operations on the given file have completed, with a failure status if needed. So far, the only implementation is a generic version for filesystems; it is, in its entirety: 

```
int generic_file_revoke(struct file *file)
        {
    	return do_fsync(file, 1);
        }
```

In the long term, `revokeat()` will need support from at least a subset of device drivers to be truly useful. 

Disconnecting access to regular file descriptors is relatively straightforward; the system call simply iterates through the list of open files on the relevant device and replaces the `file_operations` structure with a new set which returns `EBADF` for every attempted operation. (OK, for _almost_ every attempted operation - reads from sockets and device files return zero instead). The only tricky part is that it must iterate through the file list multiple times until no open files are found; otherwise there could be race conditions with other system calls creating new file descriptors at the same time that the old ones are being revoked. 

The trickier part is dealing with memory mappings. In most cases, it is a matter of finding all virtual memory areas (VMAs) associated with the file, setting the new `VM_REVOKED` flag, and calling `zap_page_range()` to clear out the associated page table entries. The `VM_REVOKED` flag ensures that any attempt to fault pages back in will result in a `SIGBUS` signal - likely to be an unpleasant surprise for any process attempting to access that area. 

Even trickier is the case of private, copy-on-write (COW) mappings, which can be created when a process forks. Simply clearing those mappings might be effective, but it could result in the death of processes which do not actually need to be killed. But it is important that the COW mapping not be a way to leak data written to the file after the `revokeat()` call. So the COW mappings are separated from each other by a simple (but expensive) call to `get_user_pages()`, which will create private copies of all of the relevant pages. 

There has been relatively little discussion of this patch so far - perhaps the relevant developers have begun their holiday breaks and revoked their access to linux-kernel. This is an important patch with a lot of difficult, low-level operations, though; that is part of why it has been so long in the making. So it will need some comprehensive review before it can be considered ready for the mainline. Given the nature of the problem, it would not be surprising if another iteration or two were needed still.
