---
title: "The Android mainlining interest group meeting: a report"
url: https://lwn.net/Articles/484196/
date: "February 28, 2012"
category: Android
author: "February 28, 2012 This article was contributed by John Stultz"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

February 28, 2012

This article was contributed by John Stultz

#### Overview and Introduction

A face-to-face meeting of the [Android mainlining interest group](<http://elinux.org/Android_Mainlining_Project>), which is a community interested in upstreaming the Android patch set into the Linux kernel, was organized by Tim Bird (Sony/CEWG) and hosted by Linaro Connect on February 10th. Also in attendance were representatives from Linaro, TI, ST-Ericsson, NVIDIA, Google's Android and ChromeOS teams, among others.

The meeting was held to give better visibility into the various upstreaming efforts going on, making sure any potential collaboration between interested parties was able to happen. It covered a lot of ground, mostly discussing Android features that recently were re-added to the staging directory and how these features might be reworked so that they can be moved into mainline officially. But a number of features that are still out of tree were also covered.

Kicking off the meeting, participants introduced themselves and their interests in the group. Also some basic goals were covered just to make sure everyone was on the same page. Tim summarized his goals for the project as: to let android run on a vanilla mainline kernel, and everyone agreed to this as a desired outcome. 

Tim also articulated an important concern in achieving this goal is the need to avoid regressions. As Android is actively shipping on a huge number of devices, we need to take care that getting things upstream at the cost of short term stability to users isn't an acceptable trade-off. To this point, Brian Swetland (Google) clarified that the Google Android developers are quite open to changes in interfaces. Since Android releases both userland and kernel bundled together, there is less of a strict need for backward API compatibility. Thus it's not the specific interfaces they rely on, but the expected behavior of the functionality provided. 

This is a key point, as there is little value in upstreaming the Android patches if the resulting code isn't actually usable by Android. A major source of difficulty here is that the expected behavior of features added in the Android patch set isn't always obvious, and there is little in the way of design documentation. Zach Pfeffer (Linaro) suggested that one way to both improve community understanding of the expected behavior, as well as helping to avoid regressions, would be to provide unit tests. Currently Android is for the most part tested at a system level, and there are very few unit tests for the kernel features added. Thus creating a collection of unit tests would be beneficial, and Zach volunteered to start that effort.

#### Run-time power management and wakelocks

At this point, the discussion was handed over to Magnus Damm to talk about power domain support as well as the wakelock implementation that the power management maintainer [Rafael Wysocki had recently posted](<https://lwn.net/Articles/479841/>) to the Linux kernel mailing list. The Android team said they were reviewing Rafael's patches to establish if they were sufficient and would be commenting on them soon. 

Magnus asked if full suspend was still necessary now that run-time power-management has improved; Brian clarified that this isn't an either/or sort of situation. Android has long used dynamic power management techniques at run time, and they are very excited about using the run-time power-management framework to further to save power, but they also want to use suspend to further reduce power usage. They really would like to see deep idle for minutes to hours in length, but, even with CONFIG_NOHZ, there are still a number of periodic timers that keep them from reaching that goal. Using the wakelocks infrastructure to suspend allows them to achieve those goals. 

Magnus asked if Android developers could provide some data on how effective run-time power-management was compared to suspend using wakelocks, and to let him know if there are any particular issues that can be addressed. The Android developers said they would look into it, and the discussion continued on to Tim and his work with the logger.

#### Logger

Back in December, [Tim posted the logger patches](<https://lwn.net/Articles/473999/>) to the linux-kernel list and got feedback for changes that he wanted to run by the Android team to make sure there were no objections. The first of these, which the Android developers supported, is to convert static buffer allocation to dynamic allocation at run-time. 

Tim also shared some thoughts on changing the log dispatching mechanism. Currently logger provides a separation between application logs and system logs, preventing a noisy application from causing system messages to be lost. In addition, one potential security issue the Android team has been thinking about is a poorly-written (or deliberately malicious) application dumping private information into the log; that information could then be read by any other application on the system. A proposal to work around this problem is per-application log buffers; however, with users able to add hundreds of applications, per-application or per-group partitioning would consume a large amount of memory. The alternative - pushing the logs out to the filesystem - concerns the Android team as it complicates post-crash recovery and also creates more I/O contention, which can cause very poor performance on flash devices. 

Additionally, the benefits from doing the logging in the kernel rather than in user space are low overhead, accurate tagging, and no scheduler thrashing. The Android developers are also very hesitant to add more long-lived daemons to the system, as while phones have gained quite a lot of memory and cpu capacity, there is still pressure to use Android on less-capable devices. 

Another concern with some of the existing community logging efforts going on is that Android developers don't really want strongly structured logging, or binary logging, because it further complicates accessing the data when the system has crashed. With plain text logging, there is no need for special tools or versioning issues should the structure change between newer and older devices. In difficult situations, with plain text logging, a raw memory dump of the device can still provide very useful information. Tim discussed a proposed filesystem interface to the logger to which the Android team re-iterated their interface agnosticism, as long as access control is present and writing to the log is cheap. Tim promised to benchmark his prototypes to ensure performance doesn't degrade.

#### ION

From here, the discussion moved to [ION](<https://lwn.net/Articles/480055/>) with Hiroshi Doyu (NVIDIA) leading the discussion. There had been an earlier meeting at Linaro Connect focusing on graphics that some of the ION developers attended, so Jesse Barker (Linaro) summarized that discussion. The basic goal there is to make sure the [Contiguous Memory Allocator (CMA)](<https://lwn.net/Articles/450286/>), [dma-buf](<https://lwn.net/Articles/474819/>), and other buffer sharing approaches are compatible with both general Linux and Android Linux. 

Dima Zavin (Google) clarified that ION is focused on standardizing the user-space interface, allowing different approaches required by different hardware to work, while avoiding the need to re-implement the same logic over and over for each device. The really hard part of buffer sharing is sorting out how to allocate buffers that satisfy all the constraints of the different hardware that will need to access those buffers. For Android, the constraints are constant per-device, so they can be statically decided, but for more generic Linux systems there may need to be some run-time detection of what devices a buffer might be shared with, possibly converting buffer types when a incompatible buffer interacts with a constrained bit of hardware for the first time. There are some conflicting needs here as Android is very focused on constant performance, and per-device optimizations allow that, whereas the more general proposed solution might have occasional performance costs, requiring buffers to be occasionally copied. 

The ION work is still under development, and hasn't yet been submitted to the kernel mailing list, but Hiroshi pointed out that a number of folks are actively working on integrating the dma-buf and CMA work with ION, thus there is a need for a point where discussion and collaboration can be done. As it seems the ION work won't move upstream or into staging immediately, it's likely the [linaro-mm-sig mailing list](<http://lists.linaro.org/mailman/listinfo/linaro-mm-sig>) will be the central collaboration point.

#### Ashmem & the Alarm driver

The next topic, was [ashmem](<http://elinux.org/Android_Kernel_Features#ashmem>), which John Stultz (Linaro/IBM) has been working on. He helped get the ashmem patches reworked and merged into staging, and is now working on pulling the pinning/unpinning feature from ashmem out of the driver and pushing it a little more deeply into the VM subsystem via the [fadvise volatile interface](<https://lwn.net/Articles/468896/>). The effort is still under development, but there have been a number of positive comments on the general idea of the feature. 

The current discussion with the code on linux-kernel is around how to best store the volatile ranges efficiently, and some of the behavior around coalescing neighboring volatile ranges. In addition there have been requests by Andrew Morton to find a way to reuse the range storage structure with the POSIX file locking code, which has similar requirements. This effort wouldn't totally replace the ashmem driver, as the ashmem driver also provides a way to get unlinked tmpfs file descriptors without having tmpfs mounted, but it would greatly shrink the ashmem code, making it more of a thin "Android compatibility driver", which would hopefully be more palatable upstream.

John then covered the Android Alarm Driver feature, which he had worked on partially upstreaming with the [virtualized RTC and POSIX alarm_timers](<https://lwn.net/Articles/429925/>) work last year. John has refactored the Android Alarm driver and has submitted it for staging in 3.4; he also has a patch queue ready which carves out chunks of the Android Alarm driver and converts it to using the upstreamed alarm_timers. To be sure regressions are not introduced, this conversion will likely be done slowly, so that any minor behavioral differences between the Android alarm infrastructure and the upstreamed infrastructure can be detected and corrected as needed. Further, the upstream POSIX alarm user interfaces are not completely sufficient to replace the Android Alarm driver, due to how it interacts with wakelocks, but this patch set will greatly reduce the alarm driver, and from there we can see if it would either be appropriate to preserve this smaller Android specific interface, or try to integrate the functionality into the timerfd code.

#### Low memory killer & interactive cpufreq governor

As Anton Vorontsov (Linaro) could not attend, John covered Anton's work on improving the low memory killer in the staging tree as well as his plans to re-implement the low memory killer in user space using a low-memory notifier interface to signal the Android Activity Manager. There is wider interest in the community in a low-memory notifier interface, and a number of different approaches have been proposed, using both control groups (with the memory controller) or simpler character device drivers. 

The Android developers expressed concern that if the killing logic is pushed out to the Activity Manager, it would have to be locked into memory and the killing path would have to be very careful not to cause any memory allocations, which is difficult with Dalvik applications. Without this care, if memory is very tight, the killing logic itself could be blocked waiting for memory, causing the situation to only get worse. Pushing the killing logic into its own memory-locked task which was then carefully written not to trigger memory allocations would be needed, and the Android developers weren't excited about adding another long-lived daemon to the system. John acknowledged those concerns, but, as Anton is fairly passionate about this approach, he will likely continue his prototyping in order to get hard numbers to show any extra costs or benefits to the user-space approach.

Anton had also recently pushed out the [Android "interactive cpufreq governor"](<https://lwn.net/Articles/480010/>) to linux-kernel for discussion. Due to a general desire in the community not to add any more cpufreq code, it seems the patches won't be going in as-is. Instead it was suggested that the P-state policy should be in the scheduler itself, using the soon-to-land average load tracking code, so Anton will be doing further research there. There was also a question raised whether the ondemand cpufreq governor would not now be sufficient, as a number of bugs relating to the sampling period had been resolved in the last year. The Android developers did not have any recent data on ondemand, but suggested Anton get in touch with Todd Poyner at Google as he is the current owner of the code on their side.

#### Items of difficulty

Having covered most of the active work, folks discussed which items might be too hard to ever get upstream.

The [paranoid network interface](<http://elinux.org/Android_Security#Paranoid_network-ing>), which hard-fixes device permissions to specific group IDs, is clearly not generic enough to be able to go upstream. Arnd Bergmann (Linaro/IBM) however commented that network namespaces could be used to provide very similar functionality. The Android developers were very interested in this, but Arnd warned that the existing documentation is narrowly scoped and will likely be lacking to help them do exactly what they want. The Android developers said they would look into it.

The binder driver also was brought up as a difficult bit of code to get included into mainline officially. The Android developers expressed their sympathy as it is a large chunk of code with a long history, and they themselves have tried unsuccessfully to implement alternatives. However, the specific behavior and performance characteristics of binder are very useful for their systems. Arnd made the good point that there is something of a parallel to SysV IPC. The Linux community would never design something like SysV IPC, and generally has a poor opinion of it, but it does support SysV IPC to allow applications that need it to function. Something like this sort of a rationale might be persuasive to get binder merged officially, once the code has been further cleaned up.

At this point, after 4 hours of discussion, the focus started to fray. John showed a demo of the benefit of the Android drivers landing in staging, with an AOSP build of Android 4.0 running on a vanilla 3.3-rc3 kernel with only few lines of change. Dima then showed off a very cool debugging tool for their latest devices. With a headphone plug on one end and USB on the other, he showed off how the headphone jack on the phone can do double duty as a serial port allowing for debug access. With this, excited side conversations erupted. Tim finally thanked everyone for attending and suggested we do this again sometime soon, personal and development schedules allowing.

#### Thanks

Many thanks to the meeting participants for an interesting discussion, Tim Bird for organizing, and to Paul McKenney, Deepak Saxena, and Dave Hansen for their edits and thoughts on this summary.
