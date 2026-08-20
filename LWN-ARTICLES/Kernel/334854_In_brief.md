---
title: In brief
url: https://lwn.net/Articles/334854/
date: "May 27, 2009"
category: recvmmsg
author: "By Jonathan Corbet May 27, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 27, 2009

**Union directories**. While a number of developers are working on the full union mount problem, Miklos Szeredi has taken a simpler approach: [union directories](<http://lwn.net/Articles/334064/>). Only top-level directory unification is provided, and changes can only be made to the top-level filesystem. That eliminates the need for a lot of complex code doing directory copy-up, whiteouts, and such, but also reduces the functionality significantly. 

**Optimizing writeback timers** : on a normal Linux system, the `pdflush` process wakes up every five seconds to force dirty page cache pages back to their backing store on disk. This wakeup happens whether or not there is anything needing to be written back. Unnecessary wakeups are increasingly unwelcome, especially on systems where power consumption matters, so it would be nice to let `pdflush` sleep when there is nothing for it to do. 

Artem Bityutskiy has put together [a patch set](<http://lwn.net/Articles/334829/>) to do just that. It changes the filesystem API to make it easier for the core VFS to know when a specific filesystem has dirty data. That information is then used to decide whether `pdflush` needs to be roused from its slumber. The idea seems good, but there's one little problem: this work conflicts with the [per-BDI flusher threads patches](<http://lwn.net/Articles/326552/>) by Jens Axboe. Jens's patches get rid of the `pdflush` timer and make a lot of other changes, so these two projects do not currently play well together. So Artem is headed back to the drawing board to base his work on top of Jens's patches instead of the mainline. 

**recvmmsg()**. Arnaldo Carvalho de Melo has [proposed a new system call](<http://lwn.net/Articles/334532/>) for the socket API: 

```
struct mmsghdr {
    	struct msghdr	msg_hdr;
    	unsigned	msg_len;
        };
    
        ssize_t recvmmsg(int socket, struct mmsghdr *mmsg, int vlen, int flags);
```

The difference between this system call and `recvmsg()` is that it is able to accept multiple messages with a single call. That, in turn, reduces system call overhead in high-bandwidth network applications. The comments in the patch suggest that `sendmmsg()` is in the plans, but no implementation has been posted yet. 

There was a [suggestion](<https://lwn.net/Articles/334855/>) that this functionality could be obtained by extending `recvmsg()` with a new message flag, rather than adding a new system call. But, as David Miller [pointed out](<https://lwn.net/Articles/334858/>), that won't work. The kernel currently ignores unrecognized flags; that will make it impossible for user space to determine whether a specific kernel supports multiple-message receives or not. So the new system call is probably how this feature will be added.
