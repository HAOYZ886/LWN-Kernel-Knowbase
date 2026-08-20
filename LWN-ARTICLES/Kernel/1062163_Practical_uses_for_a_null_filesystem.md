---
title: Practical uses for a null filesystem
url: https://lwn.net/Articles/1062163/
date: "March 12, 2026"
category: "Filesystems-nullfs; Kernel threads; Releases-7.0"
author: "By Jonathan Corbet March 12, 2026"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 12, 2026

One of the first changes merged for the upcoming 7.0 release was [nullfs](<https://lwn.net/ml/all/20260112-work-immutable-rootfs-v2-0-88dd1c34a204@kernel.org/>), an empty filesystem that cannot actually contain any files. One might logically wonder why the kernel would need such a thing. It turns out, though, that there are places where a null filesystem can come in handy. For 7.0, nullfs will be used to make life a bit easier for `init` programs; future releases will likely use nullfs to increase the isolation of kernel threads from the `init` process. 

#### Making root actually pivotable

The process of bootstrapping a computer involves a number of tricky steps, one of which is locating and mounting the root filesystem. That task might involve digging through text files, contacting other systems over the network, assembling RAID volumes, and more. As a result, there really needs to be a temporary root filesystem, with usable commands, before the real root filesystem can be mounted. That initial filesystem usually comes in the form of an initramfs image that is bundled with the kernel binary. 

That initial filesystem is the first root filesystem, but there comes a time when it must be replaced with the real root. The kernel provides a system call, [`pivot_root()`](<https://www.man7.org/linux/man-pages/man2/pivot_root.2.html>), for just that purpose. It will cause a new filesystem to become the root filesystem, and will cause any existing processes to move from the old root to the new. There is only one tiny little problem: `pivot_root()` cannot be used for this purpose for the actual root filesystem. From the man page: 

> The rootfs (initial ramfs) cannot be pivot_root()ed. The recommended method of changing the root filesystem in this case is to delete everything in rootfs, overmount rootfs with the new root, attach stdin/stdout/stderr to the new /dev/console, and exec the new init(1). Helper programs for this process exist; see switch_root(8). 

Even with the availability of a helper, the need for this kind of workaround is seen by some as mildly inelegant. (The system call _can_ be used in other contexts, such as setting up a new root for a container). 

The solution is to base the filesystem tree on an empty filesystem — nullfs — upon which both the temporary and permanent root filesystems can be mounted. `pivot_root()` can then be used to place the permanent root below the temporary one in the mount stack, allowing the latter to be unmounted. The system can then continue the bootstrap without the need for the above-described workaround. The nullfs implementation, as found in 7.0, enables this type of operation. 

#### Isolating kernel threads

There are other possible uses for nullfs, though; consider the case of kernel threads. The first two processes created by a booting Linux kernel are `init` and `kthreadd`. The `init` process will serve as the ultimate ancestor for all user-space processes created thereafter. Instead, `kthreadd` is the parent for all of the kernel threads that will be spawned over the life of the system. When the `ps` command shows you a process with a name like "`[rcu_tasks_rude_kthread]`", you'll know that it is an ill-mannered child of `kthreadd`. 

Christian Brauner (who implemented nullfs), pointed out an interesting relationship between those two processes in [this patch series cover letter](<https://lwn.net/ml/all/20260311-work-kthread-nullfs-v3-0-3dd2cbe92ad0@kernel.org>). They share the same initial `fs_struct` ~~file-descriptor table~~ , meaning that they each have access to the other's filesystem state. When `init` forks, it explicitly disconnects its children from its file table, but `kthreadd` does not do that. As a result, every running kernel thread shares full access to `init`'s filesystem state, and vice versa. Normally, all of those processes mutually trust each other, but the potential for mischief (and unfortunate bugs) is real. 

Brauner's proposed solution is to isolate kernel threads from the initial ~~file table~~ `fs_struct` and, instead, to run each of them with a nullfs instance as its root filesystem. In that way, a kernel thread can no longer interfere with the `init` process; indeed, it has no access to the filesystem at all. This seems like a bit of worthwhile kernel hardening, given that most kernel threads have no need for filesystem access. As an added bonus, `pivot_root()` no longer needs to forcibly change the root filesystem for all of the running kernel threads, since they no longer need to be moved to the new root. 

The `init` process, too, is separated from the initial `fs_struct` and given one of its own, just like any other fully independent process. At that point, the initial `fs_struct` is entirely unused — most of the time. 

Separating kernel threads from the filesystem is almost always the right thing to do; as Brauner [noted](<https://lwn.net/ml/all/20260310-schummeln-akkord-db58004d6f59@brauner>): ""Offloading fs work to kthreads is really nasty [...] It's a broken concept."" It is also unnecessary most of the time; after all, `rcu_tasks_rude_kthread` can happily go about its job of inconsiderately interrupting CPUs to force RCU grace periods without accessing any files. But there are other kernel threads that do, indeed, need occasional access to the filesystem. Enabling this access is why kernel threads have long retained their connection to the initial `fs_struct`. For cases like that, the patch series adds a mechanism to temporarily give a kernel thread access to the filesystem: 

```
scoped_with_init_fs() {
        	/* code here can perform filesystem operations */
        }
```

The use of the [scoped guard](<https://lwn.net/Articles/934679/>) ensures that the access is only provided within the indicated scope, with no possibility of that access being left in place accidentally. 

There are roughly a dozen kernel threads that have to be patched to use this new mechanism. The production of core dumps, for example, naturally requires filesystem access. Unix-domain sockets need to be able to look up names in the filesystem, firmware loading must be able to find and open the file containing the firmware, the devtmpfs filesystem must be able to provide the filesystem content, and so on. So there are a number of holes punched into the wall separating kernel threads from the filesystem, but they are small, localized, and easy to find. 

Brauner is careful to not set expectations too high for this work at this point: ""Is it crazy? Yes. Is it likely broken? Yes. Does it at least boot? Yes."" It is a significant change to some of the deepest code within the kernel, code that has had its current form for a long time. The existence of surprises seems almost certain. But, so far, nobody has questioned the goals or direction of this patch series. Greater isolation for kernel threads (and `init`) thus seems likely to show up in a future kernel release.
