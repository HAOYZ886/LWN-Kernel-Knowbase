---
title: Sysfs and a stable kernel ABI
url: https://lwn.net/Articles/172986/
date: "February 22, 2006"
category: "Development model-User-space ABI; Device model; Sysfs"
author: 
---

Some things are fairly predictable. There is a long list of regressions in the 2.6.16 kernel, and some of those do not appear to be getting a whole lot of developer attention. But when one of those bugs causes a developer's iPod to stop working with Linux, it _will_ get fixed in a timely manner. This time around, it also set off a discussion on what it really means to have a stable application interface to the kernel. 

Back in the dim and distant past (last year), the "user events" mechanism was added to the kernel. One of the first events to be implemented was block device mount and unmount operations. Over time, however, it was concluded that user events were not the right way to communicate this information. So a new interface - allowing interested user-space processes to call `poll()` on `/proc/mounts` \- was added to the kernel. Then, a patch was merged for 2.6.16 which removes the mount and unmount events. 

When Pekka Enberg (the iPod user) fingered this patch as the cause of the problem, the author of that patch (Key Sievers) responded: ""Upgrade HAL, it's too old for that kernel."" This response [didn't sit well](<https://lwn.net/Articles/172988/>) with Andrew Morton: 

You took a kernel interface which was present in 2.6.10, 2.6.11, 2.6.12, 2.6.13, 2.6.14 and 2.6.15 and changed it in a non-compatible way, without telling us that it was non-compatible and without even notifying people that we'd gone and broken existing userspace. 

We. Don't. Do. That. 

Linus, too, was [unimpressed](<https://lwn.net/Articles/172989/>): 

Guys: you now have two choices: fix it by sending me a patch and an explanation of what went wrong, or see the patch that broke things be reverted.... I'm fed up with hearing how "breaking user space is ok because it's HAL or hotplug". IT IS NOT OK. Get your damn act together, and stop blaming other people. 

For now, the issue has been resolved by reverting the patch in question. The feature removal schedule has been updated to note that the mount and unmount events will disappear in February of 2007. iPod owners can rest easy for now. 

But this episode drives home a point which is worth noting. Longstanding kernel policy has been that, while kernel internals can change at any time, the user-space interface must remain absolutely stable. Even when an interface turns out to have been badly designed, it must continue to work. Interfaces can be augmented or superseded, but they cannot be broken. 

Not that long ago, the kernel ABI consisted entirely of the system call interface and a few files in `/proc`. While regressions were not unknown, the fact is that keeping a couple hundred system calls in a stable state is a relatively straightforward task. People notice when a system call interface is changed. In more recent times, the interface to the kernel has gotten much wider; it includes several netlink-based protocols and a number of kernel-based virtual filesystems like configfs and sysfs. It can be easy for kernel developers to lose track of the fact that, when they work on one of those interfaces, they risk breaking the user-space ABI. And it can be easy for changes which change the user-space interface to slip past the review process. 

This risk is especially acute with sysfs. The directory tree exported via sysfs matches, in a very close way, the data structures maintained within the kernel. Every sysfs directory corresponds to a kobject embedded within some kernel structure, and every sysfs attribute is tied, somehow, to an attribute of the associated structure within the kernel. There are some advantages to this arrangement; sysfs has become a clear window into the organization of the system as seen by the kernel. And, because sysfs is so closely tied to the kernel's data structures, most developers need not even think about it. When a new type of device, for example, is added to the kernel, the associated sysfs entries will generally just happen by themselves. 

But every entry in sysfs - 3400 attributes in 1175 directories on your editor's relatively simple system - is part of the kernel ABI. That's 3400 attributes tied to 1175 kernel internal data structures which cannot be changed without the risk of breaking user-space code. Sysfs has evolved into a highly complex - and, to a great extent, undocumented - binary interface to the kernel. In the short term, that makes sysfs susceptible to inadvertent regressions as developers make changes without thinking about the possible user-space effects. 

In the longer term, a different problem might arise. The kernel developers have always been willing to make incompatible changes to the internal API if the end result is a better, more capable, or safer interface. This freedom to change things is widely exploited; see [the LWN 2.6 API changes page](<https://lwn.net/Articles/2.6-kernel-api/>) to see just how widely. As kernel data structures get tied into sysfs, however, they become part of an ABI which cannot be broken. In a few years, the kernel hackers may find themselves in the position of wanting to make significant internal structural changes, only to be thwarted by the inability to change the associated sysfs structure. At that point, the choice be to either (1) not make the changes, or (2) interpose some sort of compatibility translation layer between sysfs and the kernel structures it represents. Neither looks like a whole lot of fun.
