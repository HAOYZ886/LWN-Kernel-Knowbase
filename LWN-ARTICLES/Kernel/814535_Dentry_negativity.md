---
title: Dentry negativity
url: https://lwn.net/Articles/814535/
date: "March 12, 2020"
category: Dentry cache
author: "By Jonathan Corbet March 12, 2020"
---

By **Jonathan Corbet**  
March 12, 2020

Back in 2017, Waiman Long [posted a patch set](<https://lwn.net/Articles/728085/>) placing limits on the number of "negative dentries" stored by the kernel. The better part of three years later, that work continues with, seemingly, no better prospects for getting into the mainline. It would be understandable, though, if many people out there don't really know what negative dentries are or why kernel developers care about them. That, at least, can be fixed, even if the underlying problem seems to be more difficult. 

A "dentry" in the Linux kernel is the in-memory representation of a directory entry; it is a way of remembering the resolution of a given file or directory name without having to search through the filesystem to find it. The dentry cache speeds lookups considerably; keeping dentries for frequently accessed names like `/tmp`, `/dev/null`, or `/usr/bin/tetris` saves a lot of filesystem I/O. 

A _negative_ dentry is a little different, though: it is a memory of a filesystem lookup that failed. If a user types "`more cowbell`" and no file named `cowbell` exists, the kernel will create a negative dentry recording that fact. Should our hypothetical user, being a stubborn type, repeat that command, the kernel will encounter the negative dentry and reward said user — who is unlikely to be grateful, users are like that — with an even quicker "no such file or directory" error. 

Optimized error messages for fat-fingered commands is a nice benefit from negative dentries, but their real value lies elsewhere. It turns out that lookups on nonexistent files happen frequently, and it's often the same files that are being looked for. Shared-library lookups are one example; it can be instructive to type something like this: 

```
$ strace -eopenat /usr/bin/echo 'Subscribe to LWN'
```

On your editor's system, the output looks like: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

```
openat(AT_FDCWD, "/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
        openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
        openat(AT_FDCWD, "/usr/share/locale/locale.alias", O_RDONLY|O_CLOEXEC) = 3
        openat(AT_FDCWD, "/usr/lib/locale/en_US.UTF-8/LC_IDENTIFICATION", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
        openat(AT_FDCWD, "/usr/lib/locale/en_US.utf8/LC_IDENTIFICATION", O_RDONLY|O_CLOEXEC) = 3
        [...]
```

That simple `echo` command generates 13 failed lookups on a Fedora 31 system; launching `oowriter` creates 68 of them, and launching `gnucash` generates 277. For applications like these, optimizing failed lookups can yield a perceptible improvement in startup time. Compilers and language runtimes can also generate a lot of failed lookups; consider, for example, the handling of C `#include` or Python `import` statements. A quick "allmodconfig" kernel build run on your editor's system caused 52,799,262 failed lookups; that is worth optimizing. 

There is one little problem with negative dentries, though: they require memory. All of those failed lookups can generate a _lot_ of negative dentries, to the point that they start to crowd out more useful data. This is not a new problem; LWN [reported on](<https://lwn.net/Articles/1511/>) a complaint about negative dentries from memory-management developer Andrea Arcangeli — in 2002. For the most part, though, the normal shrinker mechanisms that keep the dentry cache as a whole under control have also sufficed to keep the negative variety from taking over. 

Long has been working on the cases where normal shrinking doesn't work, though; he posted [a new version of his patch set](<https://lwn.net/ml/linux-kernel/20200226161404.14136-1-longman@redhat.com/>) toward the end of February. As he points out, the number of positive dentries is limited by the number of files in the system, but there is no practical limit to the number of files that don't exist. As an illustration of what this can mean, Eric Sandeen [pointed out](<https://lwn.net/ml/linux-kernel/0e5124a2-d730-5c41-38fd-2c78d9be4940@redhat.com/>) some [code in the NSS library](<https://github.com/nss-dev/nss/blob/317cb06697d5b953d825e050c1d8c1ee0d647010/lib/softoken/sdb.c#L390>) that deliberately tries to open 10,000 nonexistent files — every time it starts up — as a timing exercise. Even without such pathological examples, though, the number of negative dentries has the potential to grow without bound. 

Long's patch set adds a new sysctl knob, `/proc/sys/fs/dentry-dir-max`; if its value is zero (the default), the system's behavior is unchanged. If, instead, it is set to a positive value, the number of negative dentries associated with any given directory will not be allowed to exceed that value. The limit on negative dentries can be no lower than 256 to avoid excessive trimming of dentries. When the time comes to clean up excess dentries, the code tries to pick those that have not been referenced recently, and will reduce the number to 7/8 of the limit. A static key is used to prevent this mechanism from slowing down the system if it is not being used. 

There seems to be no disagreement with the idea of putting firmer limits on how many negative dentries can exist. The specific solution chosen here, though, is a bit more controversial. Adding new sysctl knobs is always a bit of a hard sell; as Matthew Wilcox [put it](<https://lwn.net/ml/linux-kernel/20200226162954.GC24185@bombadil.infradead.org/>): ""A sysctl is just a way of blaming the sysadmin for us not being very good at programming"". In general, such knobs are difficult for administrators to discover in the first place, and even harder for them to set correctly. How should an administrator know what an appropriate number of negative dentries for any given directory should be for their systems and workloads? 

Thus, Wilcox and others argued for some sort of dynamic limit calculated (and adjusted) by the kernel itself. Long [responded](<https://lwn.net/ml/linux-kernel/12e8d951-fc35-bce0-e96d-f93bccf2bd3a@redhat.com/>) with a suggestion that the administrator could control the total amount of memory used by negative dentries instead of setting a per-directory maximum count; Wilcox [didn't care](<https://lwn.net/ml/linux-kernel/20200226212844.GD24185@bombadil.infradead.org/>) how the mechanism worked internally, but insisted that it had to be self-tuning. 

Dave Chinner, instead, [wondered about the need](<https://lwn.net/ml/linux-kernel/20200227083029.GL10737@dread.disaster.area/>) for this kind of mechanism at all. He suggested that the offending applications should just be confined to a memory control group; when memory gets tight within the group, the system will reclaim memory inside that group, including negative dentries. There is, he said, already an effective mechanism for limiting the amount of memory used by a specific application, so there should be no need to add another. 

Long [answered](<https://lwn.net/ml/linux-kernel/e9625cae-ee3f-3e58-903d-dabc131c8c9b@redhat.com/>) that, while control groups can help, they don't solve the entire problem. Large numbers of negative dentries can impact the performance of the program generating them, even if a control group isolates the rest of the system from the problem. He also pointed out that daemons often run in the root control group, where they cannot be constrained in this manner. 

As has happened every time that this patch set has been posted, the discussion wound down without any sort of conclusion on how things should proceed. This patch set seems no closer to the mainline than it was years ago; a search for control over negative dentries in the kernel will return a negative result.
