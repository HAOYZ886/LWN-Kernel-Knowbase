---
title: xstat() and fxstat()
url: https://lwn.net/Articles/394298/
date: "June 30, 2010"
category: "Filesystems-stat; System calls"
author: "By Jonathan Corbet June 30, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 30, 2010

POSIX has long defined variants on the `stat()` system call, which returns information about files in the filesystem. There are a couple of limitations associated with `stat()` which have been seen as a problem for a while: it can only return the information defined in the standard, and it returns _all_ of that information, regardless of whether the caller needs it. David Howells has attempted to address both problems with [a new set of system calls](<http://lwn.net/Articles/394279/>): 

```
ssize_t xstat(int dfd, const char *filename, unsigned atflag,
    	          struct xstat *buffer, size_t buflen);
    
        ssize_t fxstat(int fd, struct xstat *buffer, size_t buflen);
```

The `struct xstat` structure resembles `struct stat`, but with some differences. It includes fields for file metadata like the creation time, the inode "generation number," and the "data version number" for filesystems which support this information, and it has a version number to provide for changes in the system call API in the future. 

What also has been added, though, is a `query_flags` field where the caller specifies which fields are actually desired; if all that is needed is the file size or the link count, for example, the caller can say so. The kernel may return other information as well, but it does not have to go out of its way to ensure that it's accurate. There can be a real performance benefit to this behavior, especially for network-mounted filesystems where getting an updated value may require a conversation with the server. There is also a provision for adding "extra results" for types of metadata which are not currently envisioned. 

The addition of this sort of `stat()` variant has been discussed for years, so something is likely to be merged. Chances are good, though, that the API will change somewhat before the patch is finalized. There were [objections](<https://lwn.net/Articles/394299/>) to the use of a version number in the `xstat` structure; the overhead of supporting another system call, should one become necessary, will be less than that of dealing with multiple versions. There were also complaints about the use of that structure as both an input and an output parameter, so the input portion (the query flags) may just become a direct system call parameter instead. 

**Update** : there is already [a new version of the patch](<https://lwn.net/Articles/394391/>) available with some changes to the system call API.
