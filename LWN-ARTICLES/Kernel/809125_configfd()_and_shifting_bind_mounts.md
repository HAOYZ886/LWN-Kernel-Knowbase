---
title: configfd() and shifting bind mounts
url: https://lwn.net/Articles/809125/
date: "January 10, 2020"
category: "Filesystems-Mounting; System calls"
author: "By Jonathan Corbet January 10, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 10, 2020

The 5.2 kernel saw the addition of an extensive new API for the mounting (and remounting) of filesystems; [this article](<https://lwn.net/Articles/759499/>) covered an early version of that API. Since then, work in this area has mostly focused on enabling filesystems to support this API fully. James Bottomley has taken a look at this API as part of the job of redesigning his [shiftfs filesystem](<https://lwn.net/Articles/718639/>) and found it to be incomplete. What has followed is a significant set of changes that promise to simplify the mount API — though it turns out that "simple" is often in the eye of the beholder. 

The mount API work replaces the existing, complex `[mount()](<http://man7.org/linux/man-pages/man2/mount.2.html>)` system call with a half-dozen or so new system calls. An application would call `fsopen()` to open a filesystem stored somewhere or `fspick()` to open an already mounted filesystem. Calls to `fsconfig()` set various parameters related to the mount; `fsmount()` is then called to mount a filesystem within the kernel and `move_mount()` to attach the result to the filesystem hierarchy somewhere. There are a couple more calls to fill in other parts of the interface as well. The intent is for this set of system calls to be able to replace `mount()` entirely with something that is more flexible, capable, and maintainable. 

Back in November, Bottomley [discovered](<https://lwn.net/ml/linux-fsdevel/1574295100.17153.25.camel@HansenPartnership.com/>) one significant gap with the new API: it is not possible to use it to set up a read-only bind mount. The problem is that bind mounts are special; they do not represent a filesystem directly. Instead, they can be thought of as a view of a filesystem that is mounted elsewhere. There is no superblock associated with a bind mount, which turns out to be a problem where the new API is concerned, since `fsconfig()` is designed to operate on superblocks. An attempt to call `fsconfig()` on a bind mount will end up modifying the original mount, which is almost certainly not what the caller had in mind. So there is no way to set the read-only flag for a bind mount. 

David Howells, the creator of the new mount API, [responded](<https://lwn.net/ml/linux-fsdevel/17268.1574323839@warthog.procyon.org.uk/>) that what is needed is yet another system call, `mount_setattr()`, which would change attributes of mounts. That would work for the read-only case, Bottomley [said](<https://lwn.net/ml/linux-fsdevel/1574348414.3277.6.camel@HansenPartnership.com/>), but it falls down when it comes to more complex situations, such as his proposed UID-shifting bind mount. Instead, he [said](<https://lwn.net/ml/linux-fsdevel/1574352920.3277.18.camel@HansenPartnership.com/>), the file-descriptor-based configuration mechanism provided by `fsconfig()` is well suited to this job, but it needs to be made more widely applicable. He suggested that this interface be made more generic so that it could be used in both situations (and beyond). 

He posted [an initial version](<https://lwn.net/ml/linux-fsdevel/1574886778.21593.7.camel@HansenPartnership.com/>) of this proposed interface in November, and has recently come back with [an updated version](<https://lwn.net/ml/linux-fsdevel/20200104201432.27320-1-James.Bottomley@HansenPartnership.com/>). It adds two new system calls: 

```
int configfd_open(const char *name, unsigned int flags, unsigned int op);
        int configfd_action(int fd, unsigned int cmd, const char *key, void *value,
        			int aux);
```

A call to `configfd_open()` would open a new file descriptor intended for the configuration of the subsystem identified by `name`; the usual `open()` flags would appear in `flags`, and `op` defines whether a new configuration instance is to be created or an existing one modified. `configfd_action()` would then be used to make changes to the returned file descriptor. The `fsconfig()` system call (along with related parts like `fsopen()` and `fspick()`) is reimplemented using the new calls. Bottomley provides an example for mounting a `tmpfs` filesystem: 

```
fd = configfd_open("tmpfs", O_CLOEXEC, CONFIGFD_CMD_CREATE);
        configfd_action(fd, CONFIGFD_SET_INT, "mount_attrs", NULL,
    		    MOUNT_ATTR_NODEV|MOUNT_ATTR_NOEXEC);
        configfd_action(fd, CONFIGFD_CMD_CREATE, NULL, NULL, 0);
        configfd_action(fd, CONFIGFD_GET_FD, "mountfd", &mfd, O_CLOEXEC);
        move_mount("", mfd, AT_FDCWD, "/mountpoint", MOVE_MOUNT_F_EMPTY_PATH);
```

The `configfd_open()` call creates a new `tmpfs` instance; the first `configfd_action()` call is then used to set the `nodev` and `noexec` mount flags on that instance. The filesystem mount is actually created with another `configfd_action()` call, and the third such call is used to obtain a file descriptor for the mount that can be used with `move_mount()` to make the filesystem visible. 

With that infrastructure in place, Bottomley is able to [reimplement his shiftfs filesystem](<https://lwn.net/ml/linux-fsdevel/20200104203946.27914-1-James.Bottomley@HansenPartnership.com/>) as a type of bind mount. A shifting bind mount will apply a constant offset to user and group IDs before forwarding operations to the underlying mount; this is useful to safely allow true-root access to an on-disk filesystem from within a user namespace. 

Only one developer, Christian Brauner, has responded to this patch series so far; he [doesn't like it](<https://lwn.net/ml/linux-fsdevel/20200105162311.sufgft6kthetsz7q@wittgenstein/>). It is an excessive collection of abstraction layers, he said, and it creates another set of multiplexing system calls, a design approach that is out of favor these days: 

If they are ever going to be used outside of filesystem use-cases (which is doubtful) they will quickly rival prctl(), seccomp(), and ptrace(). That's not a great thing. Especially, since we recently (a few months ago with Linus chiming in too) had long discussions with the conclusion that multiplexing syscalls are discouraged, from a security and api design perspective. 

Unsurprisingly, Bottomley [disagreed](<https://lwn.net/ml/linux-fsdevel/1578508929.3260.61.camel@HansenPartnership.com/>). He argued that there is a common pattern that arises in kernel development: a subsystem that is complicated to configure, but then relatively simple to use. Filesystem mounts are an example of this pattern; the setup is hard, but then they can all be accessed through the same virtual filesystem interfaces. Cryptographic keys and storage devices were also mentioned. It would be better, he said, to figure out a common way of interfacing with these subsystems rather than inventing slightly different interfaces every time. The configuration file descriptor approach may be a good solution for that common way, he said: 

I don't disagree that configuration multiplexors are a user space annoyance, but we put up with them because we get a simple and very generic API for the configured object. Given that they're a necessary evil and a widespread pattern, I think examining the question of whether we could cover them all with a single API and what properties it should have is a useful one. 

The conversation appears to have stalled out at this point. It is hard to guess how this disagreement will be resolved, but one thing is fairly straightforward to point out: if the configfd approach is deemed unacceptable for the kernel, then somebody needs to come up with a better idea for how the problems addressed by configfd will be solved. Thus far, that better idea has not yet shown up on the mailing lists.
