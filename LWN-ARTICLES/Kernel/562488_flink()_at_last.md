---
title: "flink() at last?"
url: https://lwn.net/Articles/562488/
date: "August 7, 2013"
category: flink
author: "By Jonathan Corbet August 7, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 7, 2013

There has long been a desire for an `flink()` system call in the kernel. It would take a file descriptor and a file name as arguments and cause the name to be a new hard link to the file behind the descriptor. There have been concerns about security, though, that have kept this call out of the kernel; some see it as a way for a process to make a file name for a file descriptor that came from outside — via `exec()`, for example. That process may not have had a reachable path to the affected file before, so the creation of a new name could be seen as bypassing an existing security policy. 

The problem with this reasoning, as noted by Andy Lutomirski in [a patch](<http://git.kernel.org/linus/bb2314b47996491bbc5add73633905c3120b6268>) merged for 3.11-rc5, is that this functionality is already available by way of the `linkat()` system call. All it takes is having the `/proc` filesystem mounted — and a system without `/proc` is quite rare. But the incantation needed to make a link in this way is a bit arduous: 

```
linkat(AT_FDCWD, "/proc/self/fd/N", destdirfd, newname, AT_SYMLINK_FOLLOW);
```

Where "`N`" is the number of the relevant file descriptor. It would be a lot nicer, he said, to just allow the use of the `AT_EMPTY_PATH` option, which causes the link to be made to the file behind the original file descriptor: 

```
linkat(fd, "", destdirfd, newname, AT_EMPTY_PATH);
```

In current kernels, though, that option is restricted to processes with the `CAP_DAC_READ_SEARCH` capability out of the same security concerns as described above. But, as Andy pointed out, the restriction makes no sense given that the desired functionality is available anyway. So his patch removes the check, making the second variant available to all users. This functionality is expected to be useful with files opened with the `O_TMPFILE` option, but other uses can be imagined as well. It will be generally available in the 3.11 kernel.
