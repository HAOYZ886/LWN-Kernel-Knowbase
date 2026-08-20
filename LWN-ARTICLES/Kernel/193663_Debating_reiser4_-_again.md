---
title: "Debating reiser4 - again"
url: https://lwn.net/Articles/193663/
date: "August 1, 2006"
category: "Filesystems-Reiser4"
author: 
---

Hans Reiser is nothing if not persistent. Back in October, 2002, he [requested](<http://lwn.net/Articles/14152/>) that his new reiser4 filesystem be included into the 2.5 development kernel before it went into the pre-2.6 stabilization mode. Nearly four years have passed, during which reiser4 has been through endless linux-kernel debates, numerous changes to fix problems found by reviewers, the removal of core features, and a long wait in the -mm kernel. Despite all of this, reiser4 is still not in the mainline - but Hans has not given up. 

There have been a number of obstacles to overcome so far. The "files as directories" feature tweaked POSIX semantics in a way that disturbed some people, and, more importantly, had crucial locking problems; that feature has been removed. The posted benchmarks have not been entirely credible to all observers. There is concern about how committed the reiser4 developers are to ongoing support of the filesystem, once it is merged. Hans tends to have difficult relations with other kernel developers, and does not always respond entirely gracefully to (often not entirely graceful) review comments. The end result has been a difficult path toward inclusion for a filesystem which truly does offer some interesting ideas and the potential for top-level performance. 

Partially as a result of a feeling that the reiser4 process has gone on for too long, the debate has returned to linux-kernel. Hans and company would like to see reiser4 put into 2.6.19, and it seems that they might just succeed. 

Some outstanding issues remain, though some of them may not be as problematic as some people think. The biggest of those, probably, is the reiser4 plugin concept. Plugins allow the filesystem to behave differently for every file stored there; they can add features like compression, encryption, or many of the more esoteric things currently done with FUSE. Plugins raise all kinds of red flags in the development community. So, for example, Linus [states](<https://lwn.net/Articles/193673/>): 

As long you call them "plugins" and treat them as such, I (and I suspect a lot of other people) are totally uninterested, and in fact, a lot of people will suspect that the primary aim is to either subvert the kernel copyright rules, or at best to create a mess of incompatible semantics with no sane overlying rules for locking etc. 

Jeff Garzik [has concerns](<https://lwn.net/Articles/193674/>) as well: 

I don't want to be the distro support person trying to fix a crash in "reiser4", where the customer has secretly replaced the standard inode data structure with a plugin written by an intern, and secretly replaced the directory algorithm with a closed source plugin from PickYourVendor. Trying picking through that mess with a filesystem debugger. 

The message for the reiser4 developers over the last few years is that any such mechanism, if it makes sense at all, should be implemented within the VFS level, rather than within any specific filesystem. Reiser4 plugins are seen as a separate, private VFS with a long potential for problems. 

What a number of people have not realized, perhaps, is that the plugin issue is much smaller than it once might have been. They cannot be loaded at run time, so there should not be copyright issues like those that accompany closed-source kernel modules. And most of the plugin functionality has been removed in response to past comments. Andrew Morton, who has recently [reviewed the code](<https://lwn.net/Articles/193675/>) himself, comments: 

The plugins appear to be wildly misnamed - they're just an internal abstraction layer which permits later feature additions to be added in a clean and safe manner. Certainly not worth all this fuss. 

From Andrew's point of view, the biggest problems would appear to be the lack of direct I/O and extended attribute support. Direct I/O looks like it might not be too far in the future, but it does not appear that there is any immediate prospect of extended attributes. That means that, among other things, a reiser4 filesystem cannot support SELinux. That limitation may cause some distributors to leave reiser4 support out, even after reiser4 has finally been merged into the mainline kernel. 

The remaining objections may be enough to dissuade some users or distributors from working with reiser4, but it would seem that they should not be enough to block the merging of reiser4 into the mainline. A new filesystem does not affect anybody who does not use it, and the bad pitfalls for reiser4 users (deadlocks, for example) should be long gone. So it may just be that Hans Reiser's long wait is nearing its end.
