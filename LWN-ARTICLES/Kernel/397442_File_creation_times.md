---
title: File creation times
url: https://lwn.net/Articles/397442/
date: "July 26, 2010"
category: "Filesystems-Access-time tracking"
author: "By Jonathan Corbet July 26, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 26, 2010

Linux systems, like the Unix systems that came before, maintain three different timestamps for each file. The semantics of those timestamps are often surprising to users, though, and they don't provide the information that users often want to know. The possible addition of a new system call is giving kernel developers the opportunity to make some changes in this area, but there is not, yet, a consensus on how that should be done. 

The Unix file timestamps, as long-since enshrined by POSIX, are called "atime," "ctime," and "mtime." The atime stamp is meant to record the last time that the file was accessed. This information is almost never used, though, and can be quite expensive to maintain; Ingo Molnar once [called atime](<http://lwn.net/Articles/244830/>) ""perhaps the most stupid Unix design idea of all times"". So atime is often disabled on contemporary systems or, at least, rolled back to the infrequently-updated "relatime" mode. Mtime, instead, makes a certain amount of sense; it tells the user when the file was last modified. Modification requires writing to the file anyway, so updating this time is often free, and the information is often useful. 

That leaves ctime, which is a bit of a strange beast. Users who do not look deeply are likely to interpret ctime as "creation time," but that is not what is stored there; ctime, instead, is updated whenever a file's metadata is changed. The main consumer of this information, apparently, is the venerable `dump` utility, which likes to know that a file's metadata has changed (so that information must be saved in an incremental backup), but the file data itself has not and need not be saved again. The number of `dump` users has certainly fallen over the years, to the point that the biggest role played by ctime is, arguably, confusing users who really just want a file's creation time. 

So where do users find the creation time? They don't: Linux systems do not store that time and provide no interface for applications to access it. 

That situation could change, though. Some newer filesystems (Btrfs and ext4, for example) have been designed with space for file creation times. Other operating systems also provide this information, and some network filesystem protocols expect to have access to it. So it would be nice if Linux properly supported file creation times; the proposed addition of [the `xstat()` system call](<http://lwn.net/Articles/394298/>) would be the ideal time to make that change. 

Current `xstat()` implementations do, in fact, add a `st_btime` field to `struct xstat`; the "`b`" stands for "birth," which is a convention established in the BSD camp. There has been a fair amount of discussion about that addition, though, based on naming and semantics. 

The naming issue, one would think, would be relatively straightforward. It was [pointed out](<https://lwn.net/Articles/397444/>), though, that other names have been used in the kernel. JFS and Btrfs use "otime," for some reason, while ext4 uses "crtime." And BSD, it turns out, uses "birthtime" instead of "btime." That discussion inspired Linus to [exclaim](<https://lwn.net/Articles/397445/>): 

Oh wow. And all of this just convinces me that we should _not_ do any of this, since clearly it's all totally useless and people can't even agree on a name. 

After that, though, Linus looked a bit more deeply at the problem, which he saw as primarily being to provide a Windows-style creation time that Samba could use. It turns out that Windows allows the creation time to be modified, so Linus [saw it](<https://lwn.net/Articles/397446/>) as being a sort of variation on the Unix ctime notion. That led to a suggestion to change the semantics of ctime to better suit the Windows case. After all, almost nobody uses ctime anyway, and it would be a trivial change to make ctime look like the Windows creation time. This behavior could be specified either as a per-process flag or a mount-time option; then there would be no need to add a new time field. 

This idea was not wildly popular, though; Jeremy Allison [said](<https://lwn.net/Articles/397448/>) it would lead to ""more horrible confusion"". If ctime could mean different things in different situations, even fewer people would really understand it, and tools like Samba could not count on its semantics. Jeremy would rather just see the new field added; that seems like the way things will probably go. 

There is one last interesting question, though: should the kernel allow the creation time to be modified? Windows does allow modification, and some applications evidently depend on that feature. Windows also apparently has [a hack](<https://lwn.net/Articles/397449/>) which, if a file is deleted and replaced by another with the same name, will reuse the older file's creation time. BSD systems, instead, do not allow the creation time to be changed. When Samba is serving files from a BSD system, it stores the "Windows creation time" in an extended attribute so that the usual Windows semantics can be provided. 

If the current `xstat()` patch is merged, Linux will disallow changes to the creation time by default - there will be no system call which can make that change. Providing that capability would require an extended version of `utimes()` which can accept the additional information. Allowing the time to be changed would make it less reliable, but it would also be useful for backup/restore programs which want to restore the original creation time. That is a discussion which has not happened yet, though; for now, creation times cannot be changed.
