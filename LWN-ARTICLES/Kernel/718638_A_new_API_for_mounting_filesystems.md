---
title: A new API for mounting filesystems
url: https://lwn.net/Articles/718638/
date: "April 2, 2017"
category: "Filesystems-Mounting"
author: "By Jake Edge April 2, 2017 LSFMM 2017"
---

By **Jake Edge**  
April 2, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

The [`mount()`](<http://man7.org/linux/man-pages/man2/mount.2.html>) system call tries to do too many things, Miklos Szeredi said at the start of a filesystem-only discussion at LSFMM 2017. He has been interested in cleaning that up for a long time. So he wanted to discuss some ideas he had for a new interface to mount filesystems. 

`mount()` is lots of operations all rolled up into one call; there are various fields that are used in different ways depending on what needs to be done and it is almost out of flags to be used. It supports regular mounts, bind mounts, remounting, moving a mount, and changing the propagation type for a mount, but they are mutually exclusive and some operations require a remount. For example, you cannot create a read-only bind mount; you must first do the bind mount, then remount it read-only. Similarly, you cannot change the propagation parameters while doing a bind mount or changing other mount flags. 

Szeredi has come up with a proposed solution with several new system calls, starting with: 

```
int fsopen(const char *fstype, unsigned int flags);
```

It would be used to get a file descriptor to communicate with a filesystem driver and might be called as follows: 

```
fsfd = fsopen("ext4", 0);
```

That would provide a connection to the ext4 filesystem driver so that parameters could be set via a protocol. 

[ ![\[Miklos Szeredi\]](https://static.lwn.net/images/2017/lsfmm-szeredi-sm.jpg) ](<https://lwn.net/Articles/718641/>)

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The talk of a protocol prompted Jeff Layton to ask about using a netlink socket instead. But Al Viro said that a netlink protocol would need to be fully specified right from the start, which would not fit well. Josef Bacik said that he thought netlink would allow adding new attributes and values after the fact. There could a different protocol specification for each filesystem type, perhaps based on a common set for all filesystems with extensions for specific types. Layton agreed but said the mechanism for the protocol could be determined at a later point. 

The protocol Szeredi is envisioning would have a set of configuration commands, each with a NUL-delimited set of parameters. It might look something like: 

```
SETDEV\0/dev/sda1
        SETOPTS\0seclabel\0data=ordered
        ...
```

That data would be written to the filesystem file descriptor returned from `fsopen()`. 

Jeff Mahoney asked if there was a need for a system call at all. Perhaps sysfs or the `/proc` filesystem could be used instead. One attendee pointed out that would mean that some other mechanism would need to be used to mount `/proc` or `/sys`. There might also be implications for booting, since those filesystems may not be available early enough to mount the boot partition. 

Additional system calls would be needed, Szeredi said, moving back to his proposed interface. Attaching a filesystem to a mount point would be done with `mountat()`, changes to a mount would done using `mountupdate()`, while `mountmove()` to move a mount and `mountclone()` to clone one round out the set. There were suggestions that some of those could be combined into one call, `mountmove()` and `mountclone()` in particular. 

Szeredi said that he would look into using a netlink socket rather than `fsopen()`. One attendee said that netlink would make the simple case of a straightforward mount more complicated, but Szeredi said that the existing `mount()` would not be going away. 

David Howells wondered if netlink was an optional kernel component; if so, mounting using this new mechanism would be impossible in some cases, another attendee said. But, again, Szeredi said that the existing `mount()` system call could be used. There was some concern that filesystems will come to depend on these new interfaces, so that using `mount()` won't work well. 

Layton noted that there have been requests for better error messages from mounting operations; often there is not enough detail in the error code returned. Szeredi said that more detailed information could potentially be read from the descriptor returned by `fsopen()`. 

Overall, the attendees seemed interested in having a better API for mounting filesystems, but it would seem there is a ways to go before there is something concrete to ponder.
