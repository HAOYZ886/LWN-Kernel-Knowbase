---
title: Module unloading in a reference counted world
url: https://lwn.net/Articles/68149/
date: "January 27, 2004"
category: Modules; Race conditions
author: 
---

Increasingly, the kernel uses reference counts to know when data structures are no longer needed and can be reclaimed. This reference counting tends to be managed by the [kobject](<https://lwn.net/Articles/51437/>) type, though other mechanisms are used as well. When properly used, this mechanism works well. Interesting issues can come up, however, when reference-counted objects are maintained by code in loadable modules. In many situations, the module cannot be unloaded until all objects it has created have seen their reference counts go to zero and have been returned to the system. Otherwise, the system can be left with objects containing invalid references to module code which no longer exists. Bad things usually result from that situation. 

Alan Stern recently ran into this sort of situation; his module registers various structures with the device model, and must be sure not to allow itself to be unloaded until those structures have been released. To that end, he wrote [a patch](<https://lwn.net/Articles/68153/>) adding two functions (`class_device_unregister_wait()` and `platform_device_unregister_wait()`) which unregister those structures and explicitly wait until they have been released. This patch did not get very far, however; it was quickly pointed out that, with this code, it is relatively easy to deadlock the kernel. If the process trying to remove the module also has an open file descriptor to one of that module's sysfs entries, everything comes to a halt. The suggested solution, instead, is to simply not allow the module to be unloaded if it still has unreclaimed objects outstanding. 

That approach is taken in some other contexts. The `cdev` structure used to represent char devices uses a kobject for its reference count. The `cdev_get()` function does more than just increment the count in the kobject, however; it also increments the reference count for the module which drives that device. If any `cdev` structure owned by a module has references, the module, too, will have a non-zero reference count and will not be unloadable. 

Another approach has been taken in the network subsystem. The `net_device` structure represents a network device; its rules say that it must be allocated dynamically, with `alloc_netdev()`. When the network driver is done with the structure, it calls `free_netdev()` to get rid of it. The `net_device` structure has its own reference count, but it is not tied to the underlying module's reference count. Instead, the networking system guarantees that, once `free_netdev()` has been called, it will not call into the module again for that device. The release function for the `net_device` structure, which returns its memory to the system, lives in the networking code, rather than in any loadable module. As a result, the module can be removed even while some of its `net_device` structures continue to exist, and all will be well. Those structures have been detached from the module which created them, and will be freed by core kernel code. 

The real lesson from all this, perhaps, is that the kernel developers are still figuring out the implications of the lifetime rules of the objects they create. The addition of sysfs in 2.5 has tended to force this issue; sysfs exposes a great many internal kernel objects to user space, which can keep references to those objects for an indeterminate period of time. Making everything work safely in this environment has proved to be a challenge at times. 

And module unloading, of course, will always be a challenge. There will likely always be issues involved with removing code from a live kernel. [As Linus put it](<https://lwn.net/Articles/68182/>): 

The proper thing to do (and what we _have_ done) is to say "unloading of modules is not supported". It's a debugging feature, and you literally shouldn't do it unless you are actively developing that module. 

Experience shows that many users are not happy with a kernel which cannot unload modules, however. So the kernel developers are likely to be wrestling with these issues for some time yet.
