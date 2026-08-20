---
title: Freezing filesystems and containers
url: https://lwn.net/Articles/287435/
date: "June 25, 2008"
category: "Filesystems; Virtualization-Containers"
author: "By Jake Edge June 25, 2008"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
June 25, 2008

Freezing seems to be on the minds of some kernel hackers these days, whether it is the northern summer or southern winter that is causing it is unclear. Two recent patches posted to linux-kernel look at freezing, suspending essentially, two different pieces of the kernel: filesystems and containers. For containers, it is a step along the path to being able to migrate running processes elsewhere, whereas for filesystems it will allow backup systems to snapshot a consistent filesystem state. Other than conceptually, the patches have little to do with each other, but each is fairly small and self-contained so a combined look seemed in order. 

Takashi Sato [proposes](<http://lwn.net/Articles/287242/>) taking an XFS-specific feature and moving it into the filesystem code. The patch would provide an `ioctl()` for suspending write access to a filesystem, freezing, along with a thawing option to resume writes. For backups that snapshot the state of a filesystem or otherwise operate directly on the block device, this can ensure that the filesystem is in a consistent state. 

Essentially the patch just exports the `freeze_bdev()` kernel function in a user accessible way. `freeze_bdev()` locks a file system into a consistent state by flushing the superblock and syncing the device. The patch also adds tracking of the frozen state to the `struct block_device` state field. In its simplest form, freezing or thawing a filesystem would be done as follows: 

```
ioctl(fd, FIFREEZE, 0);
    
        ioctl(fd, FITHAW, 0);
```

Where fd is a file descriptor of the mount point and the argument is ignored. 

In another part of the patchset, Sato adds a timeout value as the argument to the `ioctl()`. For XFS compatibility--though courtesy of a patch by David Chinner, the XFS-specific `ioctl()` is removed--a value of 1 for the pointer argument means that the timeout is not set. A value of 0 for the argument also means there is no timeout, but any other value is treated as a pointer to a timeout value in seconds. It would seem that removing the XFS-specific `ioctl()` would break any applications that currently use it anyway, so keeping the compatibility of the argument value 1 is somewhat dubious. 

If the timeout occurs, the filesystem will be automatically thawed. This is to protect against some kind of problem with the backup system. Another `ioctl()` flag, `FIFREEZE_RESET_TIMEOUT`, has been added so that an application can periodically reset its timeout while it is working. If it deadlocks, or otherwise fails to reset the timeout, the filesystem will be thawed. Another `FIFREEZE_RESET_TIMEOUT` after that occurs will return `EINVAL` so that the application can recognize that it has happened. 

Moving on to containers, Matt Helsley posted a [patch](<http://lwn.net/Articles/287241/>) which reuses the software suspend (swsusp) infrastructure to implement freezing of all the processes in a [control group](<http://lwn.net/Articles/256389/>) (i.e. cgroup). This could be used now to checkpoint and restart tasks, but eventually could be used to migrate tasks elsewhere entirely for load balancing or other reasons. Helsley's patch set is a forward port of work originally done by Cedric Le Goater. 

The first step is to make the freeze option, in the form of the `TIF_FREEZE` flag, available to all architectures. Once that is done, moving two functions, `refrigerator()` and `freeze_task()`, from the power management subsystem to the new `kernel/freezer.c` file makes freezing tasks available even to architectures that don't support power management. 

As is usual for cgroups, controlling the freezing and thawing is done through the cgroup filesystem. Adding the `freezer` option when mounting will allow access to each container's `freezer.state` file. This can be read to get the current freezer state or written to change it as follows: 

```
# cat /containers/0/freezer.state
        RUNNING
        # echo FROZEN > /containers/0/freezer.state
        # cat /containers/0/freezer.state
        FROZEN
```

It should be noted that it is possible for tasks in a cgroup to be busy doing something that will not allow them to be frozen. In that case, the state would be `FREEZING`. Freezing can then be retried by writing `FROZEN` again, or canceled by writing `RUNNING`. Moving the offending tasks out of the cgroup will also allow the cgroup to be frozen. If the state does reach `FROZEN`, the cgroup can be thawed by writing `RUNNING`. 

In order for swsusp and cgroups to share the `refrigerator()` it is necessary to ensure that frozen cgroups do not get thawed when swsusp is waking up the system after a suspend. The last patch in the set ensures that `thaw_tasks()` checks for a frozen cgroup before thawing, skipping over any that it finds. 

There has not been much in the way of discussion about the patches on linux-kernel, but an ACK from Pavel Machek would seem to be a good sign. Some [comments](<https://lwn.net/Articles/287500/>) by Paul Menage, who developed cgroups, also indicate interest in seeing this feature merged.
