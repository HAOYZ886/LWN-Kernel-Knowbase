---
title: Bringing Android closer to the mainline
url: https://lwn.net/Articles/472984/
date: "December 20, 2011"
category: Android
author: "By Jonathan Corbet December 20, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 20, 2011

The agenda for the [2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) did not include Android as a topic, but Android came up anyway. In a conclusion that surprised many, the group agreed that the bulk of the Android kernel code should probably be merged into the mainline. The past couple of years have made it clear that Android will not be going away; it has, in particular, done a good job of outlasting the resistance to merging its code. After the Summit things got quiet again on the Android front, but that does not mean that nothing has been happening. 

Tim Bird recently [announced](<https://lwn.net/Articles/472987/>) the existence of the Android mainlining project, an effort intended to help coordinate the various groups that have been working in this area. The project has the obligatory [wiki](<http://elinux.org/Android_Mainlining_Project>) and [mailing list](<https://lists.linuxfoundation.org/mailman/listinfo/ce-android-mainline>). The list is new and has not seen a whole lot of traffic - a situation which may well change in the near future. 

Toward the end of November, the core Android code was returned to the staging tree, from which it had been removed at the end of 2009. Since the code's return to staging, changes have been going in and the code has caught up to its state in the Android tree. The code has now reached a point where, as [summarized](<https://plus.google.com/111049168280159033135/posts/5WfyJ1GAFXM>) by Greg Kroah-Hartman on December 16: 

[T]he next linux-next Linux kernel release should almost boot an Android userspace, we are only missing one piece, ashmem, and that should hopefully land in my staging-next tree next week. The patches are still being tested and cleaned up by others. 

Between the wiki and a look at [`drivers/staging/android` in linux-next](<https://git.kernel.org/?p=linux/kernel/git/next/linux-next.git;a=tree;f=drivers/staging/android;h=cf078509662793be77b206b700db429facb61c1c;hb=HEAD>), one can get a fair idea of the state of the various patches. One notable patch that is _not_ there is wakelocks (or "suspend blockers"), a feature which has been at the core of the controversy around the Android code. The wakelock concept will almost certainly return at some point, but much of the focus seems to be on the easier components at the moment. As Greg noted, wakelocks are not actually needed to boot an Android system - they're just necessary to keep that system from draining the battery too quickly. 

The pieces that exist in the linux-next staging directory now are: 

  * [Binder](<http://elinux.org/Android_Binder>), the interprocess communication mechanism used within Android. Binder could conceivably be replaced with a standard IPC mechanism or, perhaps, with D-bus, but it has a number of unique features (zero-copy message transmission, thread management, credential passing) that are hard to replace in a straightforward manner. (See [this article](<https://lwn.net/Articles/466304/>) for a detailed look at various Linux IPC mechanisms, binder included). 

  * Logger is the kernel piece of the [Android logging system](<http://elinux.org/Android_Logging_System>). It implements a completely separate path for Android-specific log messages, which do not mix with normal kernel messages in any way. Other than adding a "facility" concept to kernel logging, it's not clear what this component offers, but it is also relatively self-contained and should not be too controversial. 

  * The "low memory killer" implements Android's interesting approach to application management. In the Android world, applications never choose to exit. They hang around until memory gets tight, at which point kernel starts to kill them off. It's [a small piece of code](<https://git.kernel.org/?p=linux/kernel/git/next/linux-next.git;a=blob;f=drivers/staging/android/lowmemorykiller.c;h=4098bbb5a0c1a1eef4f0aea8bf062597a61da25a;hb=HEAD>) that works using the "shrinker" mechanism, a standard way to register functions to be called when the kernel would like to free up some memory. So, even though it is memory-management code, it is relatively unintrusive and will not affect systems where it is not used. 

  * "Pmem" is Android's answer to the age-old problem of allocating large, physically-contiguous buffers after the system has been running for a while. It works in the usual way: a range of memory is set aside at boot time. One difference with pmem is that it exports a device to user space, allowing buffers to be allocated directly by applications and passed to drivers. That, in turn, leads to things like camera drivers being written with the assumption that user space can give them physically-contiguous buffers for video frames, something that would not be possible in a mainline kernel. 

Approaches like [CMA](<https://lwn.net/Articles/447405/>) seem like a better solution to this particular problem - if and when CMA is merged into the mainline. Meanwhile, however, applications have been written using pmem, so that interface is unlikely to go away in the near future. 

  * The "RAM console" saves log data to a special region of memory where it can be found and recovered after a reboot. It is a debugging tool. 

  * "Timed GPIO" is a simple mechanism whereby the kernel can schedule a specific setting for a GPIO line at some point in the future. An example use would be to ensure that the vibrator gets turned off regardless of what happens to the application that turned it on. 

The "ashmem" component was not in linux-next as of this writing, but, as Greg noted, its arrival there is expected in the near future. Ashmem is a shared memory mechanism that is able to discard some or all of its contents when memory pressure gets high. It could conceivably be replaced by the proposed [POSIX_FADV_VOLATILE](<https://lwn.net/Articles/468896/>) operation, but the latter does not, yet, seem to be a complete solution for Android's requirements. 

There are a number of Android-specific changes that do not appear on that list, and, thus, are not likely to be merged into the mainline in the near future. Some of them are so Android-specific that they may never get in; the "network security" tweaks fall into that category. Others, such as the alarm timer code, may be superseded by enhancements in the mainline. Then, of course, there is a long list of drivers for hardware found on Android devices. Quite a few of those drivers have found their way into the mainline already, and others are on their way. 

In summary: if all goes well, the 3.3 kernel should see the delta between Android kernels and the mainline go down considerably. That should make life easier for developers and for vendors wanting to provide Android-compatible hardware. Of course, it would be unsurprising if Android were to grow new subsystems of its own in the future; the Android developers have made it clear that they are unable and unwilling to wait for the mainlining process to run its course when they have products to ship. But, with any luck at all, the worst days of a significant fork that has caused a fair amount of ill will and difficult discussion should soon be behind us.
