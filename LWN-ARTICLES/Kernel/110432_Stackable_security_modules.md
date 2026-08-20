---
title: Stackable security modules
url: https://lwn.net/Articles/110432/
date: "November 10, 2004"
category: "Modules-Security modules; Security-Security modules"
author: 
---

The Linux security module framework allows the flexible loading of security modules into the kernel. These modules are allowed to hook into a large number of kernel functions and, if they deem it appropriate, block an attempted user-space operation. As a way of helping security modules, many core kernel structures include a `void *` "security" pointer which may be used to attach security-related information. These structures include those representing inodes, files, open sockets, processes, and more. 

One shortcoming of the security module mechanism - according to some developers, at least - is that it makes life hard for people who are trying to load more than one module. There is some rudimentary support for stacking modules; essentially, any modules which request stacked loading are simply passed to the "primary" module. The primary module can refuse to accept the stacked module at all (in which case the load fails), or it can, in its own way, arrange to call the stacked module's hooks when it sees fit. So stacking a module requires that the author of the first-loaded module explicitly thought about and coded support for that mode of operation. Since that support must be added to a large number of security hooks, most security module authors conclude that they have better things to do with their time. 

There is also the little matter of that `void *` security pointer in all those structures. If modules are to be stacked, they must come up with some way of sharing that single pointer without creating chaos. 

Serge Hallyn has been trying to address the stacking problem for some time; [his latest attempt](<https://lwn.net/Articles/109712/>) was recently posted to linux-kernel with a request for comments. He certainly got a few of those. 

The patch supports stacking security modules by separating them from each other to the greatest extent possible. The existing security hooks are all set to a set of "stacker" hooks; each one calls the associated hook provided by each stacked module, and returns a failure code if any of the modules decides to block the operation. The various `void *` pointers are each replaced by a static array, dimensioned to the maximum configured number of security modules (four by default). Each loaded module is given an index into the array, and is expected to work with its entry only. Thus, all security modules must be changed to work properly in the stacking mode. 

The code itself has drawn a few complaints; not everybody is convinced by how the locking works, for example. Adding static arrays to heavily-used kernel data structures (such as files and inodes) will significantly increase kernel memory usage. Your editor, in his reading of the patch, can find no code which prevents loading more than the configured maximum number of modules and corrupting all of those structures. And so on. 

The real issue of contention, however, is whether security module stacking makes any sense in the first place. Stacked modules operate without any awareness of each other, but could interact to produce surprising results. In the security world, surprising results tend not to be welcome. The right approach, as [expressed by James Morris](<https://lwn.net/Articles/110443/>) (and others), is to load SELinux and let it handle the loading of other security policies. SELinux was designed to do this, and it should be able to handle module interactions in a more predictable way. Whether other developers are willing to accept SELinux as the One True Base Security Module remains to be seen; it seems more likely than getting blind security module stacking into the kernel, however.
