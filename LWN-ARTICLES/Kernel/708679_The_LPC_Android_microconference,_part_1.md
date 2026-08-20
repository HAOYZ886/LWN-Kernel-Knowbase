---
title: "The LPC Android microconference, part 1"
url: https://lwn.net/Articles/708679/
date: "December 14, 2016"
category: Android
author: "December 14, 2016 This article was contributed by John Stultz and Sumit Semwal Linux Plumbers Conference"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

December 14, 2016

This article was contributed by John Stultz and Sumit Semwal 

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2016-Linux_Plumbers_Conference>)

The Linux Plumbers Android microconference was held in Santa Fe on November 3rd; it had a large number of topics to cover, many in the form of lightning talks, trying to expose developers to a number of recent developments connected to Android. There was strong attendance from community members as well as from Google, including developers working on Android, Chrome OS, and Brillo. This article is the first in a two-part series describing the conversations that took place during this microconference. 

Note that the topics covered elsewhere in LWN ([recent and future scheduler efforts](<https://lwn.net/Articles/706374/>) and [Brillo kernel maintenance](<https://lwn.net/Articles/706597/>)) will not be covered here. 

#### Android code in staging

The first speaker was John Stultz, who quickly covered the current state of the Android code in the staging directory ([slides [PDF]](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3717/original/Android%20patches%20in%20Staging.pdf>)). With around 150 patches applied in the last year, it hasn’t seen a huge amount of change recently — but that is not a trivial patch rate either. Highlights were recent progress with upstreaming the Android Sync framework and the removal of the [timed output and timed GPIO drivers](<http://elinux.org/Android_Kernel_Features#timed_output_.2F_timed_gpio>). One interesting item was that no new features have been added to staging in the last year. 

The remaining Android code in staging has come down to three drivers: ashmem, the low-memory killer, and ION. Efforts to upstream ashmem’s memory unpinning feature have stalled, so input was requested on whether we should upstream it as-is (as was done with binder) or try instead to add shrinker-based unpinning to the [memfd](<https://lwn.net/Articles/593918/>) feature. 

Android has already integrated a user-space based low-memory killer utilizing memory-pressure notifiers. However, due to the high cost of calculating who to kill in user space once the notification has happened, this implementation has not shipped on any devices as of yet, leaving most devices still making use of the in-kernel low-memory killer. Discussion on the [ION memory allocator](<https://lwn.net/Articles/480055/>) was deferred to Laura Abbott’s session later in the day. 

John left the session with the question of whether staging has been particularly useful to upstreaming Android functionality, and if folks thought more features (providing some specific examples) should be pushed via staging or not. 

#### Background updates

Rom Lemarchand from Google gave a lightning talk on Android background updates ([slides [PDF]](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3879/original/Android%20Background%20Updates%20-%20LPC%202016.pdf>)). Before starting his talk, he pulled out a phone which he had avoided applying a recent update to. He hit the "update" button and began his talk while holding the phone up. His purpose was to show how much faster and less painful the update process can be. 

While the update progressed, he provided an overview of the simplified (from the user’s perspective) background-update process, then contrasted it with the complex state machine needed between the bootloader and kernel in order to allow this process to work. The android background update was adapted from Chrome OS; it uses a dual partition layout, so there must be duplicate partitions for anything updated in an over-the-air update (usually the boot and system partitions). This additional storage cost was mitigated by using squashfs and minimizing the number of applications available over the Play Store that are stored on the system partition. The Android developers also introduced a new layer to handle the communications between the kernel and the bootloader. 

There are a few additional kernel patches needed for ramfs booting and dm-verity validation for the rootfs. The system will fail back to the old partition if there’s a validation failure, so the end result is an update mechanism that lowers the chance of bricking a device. Additionally, Android now does the application rebuilding step in the background, so one doesn’t have to wait for that after installing an update. Of course, before he had finished his talk, his phone had already rebooted into the updated image and was ready to use. 

#### vDSO issues

Next up was Kevin Brodsky from ARM on the vDSO on arm64 ([slides [PDF]](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3711/original/LPC_vDSO.pdf>)). Kevin outlined how a vDSO is a shared library for user space that is provided by the kernel; it allows certain kernel system calls like `clock_gettime()`, to be processed entirely in user space, avoiding context switches and greatly improving performance. He provided a detailed look at how the vDSO is implemented before diving into his work implementing 32-bit vDSO support for arm64. 

While the performance of 32-bit apps in a 64-bit environment might seem unimportant, there are many Android apps that have not been built for 64-bit systems and may not be updated in the near future, as moving to 64 bits comes with its own costs (pointer sizes double, etc) that can negatively affect performance. Additionally, Chrome OS on arm64 platforms currently runs a 32-bit-compiled user space. So performance for 32-bit apps is important for many common use cases. 

He outlined some of the details of his proposed patches, including complications like the fact that one has to have a 32-bit toolchain installed to build the 32-bit vDSO when building an arm64 kernel, as well the need to have support added to the C library implementations in order to enable the vDSO which, unfortunately, didn’t make it for the Android Nougat release. He then covered some performance results, and again pointed to his proposed patch set submitted upstream. 

#### Kernel behavioral analysis

Next up was a talk on kernel behavioral analysis and visualization by KP Singh from Google ([slides [PDF]](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3705/original/Linux%20Behaviour%20Analysis.pdf>)). KP covered a number of tools he has been using and working on. The first was [Linux Integrated Systems Analysis (LISA)](<https://github.com/ARM-software/lisa>) which has been heavily used by ARM to automate running workloads and collecting traces. He covered [Trace Analysis And Plotting in Python (TRAPpy)](<https://arm-software.github.io/trappy/>), which parses trace files and allows for visualization. Then finally there is the [Behavioral Analysis and Regression Toolkit (BART)](<http://arm-software.github.io/bart/>), which allows expected behaviors to be defined and then tested against the trace logs, allowing for behavior regressions to be caught. 

Defining behaviors in BART is done by by first establishing a test case, such as one that generates a number of threads that run for 10% of the time. Then, if one expects such lightweight tasks to be bound to the "little" CPUs in a system, one can specify that the residency of those tasks should be on the "little" cluster for 75% of the time. He presented example charts generated from TRAPpy that showed CPU usage across the system; it illustrated the small tasks being properly packed onto the little CPUs, keeping the big CPUs idle. He then showed how BART would view that data and validate the earlier statement made as true. BART has a number of ways of testing scheduler behavior, allowing assertions about temperature and load averages to be made. 

He plugged his phone into his laptop and did a live demo, running tests using LISA, which can be used via a web page. LISA generated interactive TRAPpy graphs, also shown in that web page, as well as various BART assertions that evaluated to true or false. This generated a fair amount of discussion around the [Jupyter](<http://jupyter.org/>) web-interface used, which seemed new to many in the crowd. While the tooling can be used directly in the shell, it was noted that Jupyter Python notebooks are common in academic research, and these rich tools are useful for the type of performance analysis used in the examples. 

There were some questions about the limits of doing browser-based interactive trace charts, but those limits are mostly a function of the amount of memory on the host rendering the web page. There was also interest in how other test frameworks might be able to use these tools, one idea being that kselftests might also be able to adapt some of these detailed behavioral tests as a way to watch for regressions. 

#### Large pages in ION

The first graphics-focused discussion was about large pages in ION, by John Reitan from ARM ([slides [PDF]](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3687/original/CPA%20-%20Plumbers%202016.pdf>)). He covered some of his efforts to prototype a compound-page heap in ION. On some display and IOMMU hardware, 2MB pages are needed for rotation. As the kernel's native page size is 4K, this could previously only be done by using a special carve-out heap to guarantee allocations using 2MB of contiguous physical pages. His solution, which is designed to replace the carve-out heap usage, maintains a pool of pre-allocated and zeroed 2MB pages, which are asynchronously filled. One optimization he was hoping to achieve was to avoid internal fragmentation of the allocations, as the 2560x1440 buffers he was using only need 14.065 MB, causing only 64KB of the the last 2MB page to be used. Packing tails wasn’t an option as the memory does have to be virtually contiguous, but he could use that extra space to store other small allocations. 

He prototyped this on a Nexus 10 tablet to better represent how his allocator would work when there are numerous other device drivers active and doing things like pinning memory. The Nexus 10, while old now, had a world-leading high-resolution display at the time of its release, so it's still relevant to mid-range devices today. When initially developed, John explained, the Nexus 10 had a number of issues with the GPU pinning 4K pages, which was worked around by reserving a large 348MB carve-out for the GPU. Also, the IOMMU would cause display corruption resulting from TLB misses when enabled, which using 2MB pages avoids. So another 256MB carve-out was set aside to make that work. 

His compound page heap is able to replace these two carve-outs and can be used for all ION allocations including camera and video-decoding buffers, freeing up more than half a gigabyte of reserved memory. He ran this code through a number of scripted tests to stress the system. The system was overall more responsive, but unfortunately, he found that, after a week or two, the tests would fail and the device would not wake up. This ended up being due to failed allocations, as the GPU was pinning pages and fragmenting the new heap. So, for the next steps, he wants to improve the GPU driver to support page migration and avoid causing fragmentation. 

He discussed his plans to try to enable ION to support order-9 allocations and trying to upstream the heap code. He also wants to follow the efforts on the new Unix device memory allocator that was [discussed during XDC 2016](<https://lwn.net/Articles/703749/>). 

#### ION

Next was Laura Abbott from Red Hat covering the status of ION ([slides [PDF]](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3759/original/Ion%20@%20LPC%202016.pdf>)). Laura was mostly wanting to have a discussion around the goals and needs folks have for this allocator. ION has lingered in staging for a long time, and while it's been getting patches for things like a new query `ioctl()` and cleaning up dead code, there has been little progress toward addressing its fundamental issues. There’s unfortunately no one who is working on ION full time and, most problematically, the users of ION are, for the most part, proprietary out-of-tree graphics drivers, so there is no real reference platform that the community has any interest in. 

Laura thinks that the current iterative approach to fixing ION in the hope of upstreaming it isn’t working and should be given up. The ION driver as it exists today is not self-contained like binder, so its not really something that could be upstreamed as-is. Her proposal is to freeze ION development to try to force development focus elsewhere. She said she would still take bug fixes but, since upstreaming ION isn’t really a goal in itself, we really need to find a good solution. 

She also pointed to the new Unix device memory allocator, which is trying to address some of the same issues. This brought up some discussion in the room around the somewhat old debate of ION/Gralloc style user-space constraint awareness vs in-kernel constraint solving, as was proposed in the attach-and-allocate-at-mapping efforts made previously. 

Another discussion point was whether there is a need for a centralized dma_buf allocator. Both in-kernel constraint solving and centralized allocation were previously proposed with [cenalloc](<https://lwn.net/Articles/615892/>). Currently, the upstream approach requires user space to allocate buffers using driver-specific `ioctl()` calls; if there’s a constraint on those buffers, user space has to do the allocation from the most constrained driver. This seems to work for upstream developers, so these new approaches don’t appear to have much immediate benefit, but it was interesting to hear how folks looking at the new Unix device memory allocator for some more advanced use cases might be "rediscovering" some of the issues that ION was created to solve. 

#### Hardware composer and drm_hwcomposer

Next on the schedule was Greg Hackmann, Sean Paul, and Zach Reizner from Google talking about Hardware Composer 2.0 (HWC2) and the drm_hwcomposer implementation ([slides [PDF]](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/4185/original/LPC%20HWC%202.0%20&%20drm_hwcomposer%20.pdf>)). Greg started by providing an overview of what the hardware composer does: compositing a number of different buffers in the most efficient way possible. It is mostly used to save power by doing the composition using the display hardware's support for multiple planes and overlays, rather then compositing the buffers in the GPU (which is less power-efficient). He then reviewed some of the changes from the original HWC framework to HWC2, which entails a richer API with some function name clarifications and the use of non-speculative fences instead of the speculative ones used before. He then showed some diagrams illustrating the differences between how the APIs are used. He pointed to the implementation and documentation that is in the AOSP source, and promised a reference implementation for the Nexus 9 device soon. 

Sean and Zach discussed their efforts on drm_hwcomposer, which is a kernel modesetting-based HWC implementation that was initially done for the Pixel C tablet. They showed how the HWC2 framework simplifies some of the implementation details and got into some complexities of using overlays on some hardware. It turns out that, while doing composition in the display hardware is more power-efficient than using the GPU for screens where there isn’t much change, the in-display composition is still re-done over and over, which consumes memory bandwidth. In that case, doing composition in the GPU once can save that bandwidth. To balance this tradeoff, the screen is broken up into rectangular regions separating parts that are changing from parts that are mostly static. Then only the changing portions are sent in via the overlay. 

They outlined their GL compositor, which uses a shader per layer, allowing each rectangular region to be generated with a single drawing call. For the most part, this implementation is generic, but they have some device-specific logic in the planner code, which is used every time composition changes; it helps map SurfaceFlinger layers to hardware planes. There was some discussion at that point that having hardware-specific planners isn’t great, but it's better to have a mostly generic drm_hwcomposer with small, device-specific planners rather than having a bunch of forked drm_hwcomposers for each device. 

#### Mainlining Sync

Gustavo Padovan from Collabora was up next to talk about his efforts to move the Android Sync Framework out of the staging directory ([slides [PDF]](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/4059/original/Plumbers-Explicit-Fencing_Talk.pdf>)). He started by providing an overview of how the Android Sync framework works; it provides the concept of a timeline, which is a monotonic counter to which sync points can be added. There can be multiple timelines in the system at one time. One or more of those sync points can be merged together into what Android calls a "sync fence". These sync fences can be exported and shared in user space via file descriptors. User space can wait on those sync fences, which will be signaled when all the sync points contained have been signaled by the monotonic counter reaching their value. 

Gustavo covered his upstreaming efforts to bring explicit fencing to mainline. He removed the sync points and timelines. replacing them with the DRM fence implementation, and reworked the sync fence implementation, renaming it to "sync file". He has also implemented patches to Android’s libsync library to be able to support both legacy Android sync fences as well as sync files. He clarified the types of fences DRM uses, which can be "in fences", which are fences that can be waited on, and "out fences", which are per-CRT and are signaled on every scanout, informing the user that the previous buffer can be reused. Support for explicit fencing must be added to each driver and, at this point, freedreno is done and work is in progress on i915 and virgl. User-space support is also needed in mesa, which is in progress by Rob Clark. drm_hwcomposer already supports explicit fencing; it has been a good test case for the upstream efforts, but work there continues. 

There were some questions as to what benefits explicit fencing brings over implicit fencing, and it was clarified that the performance should be better. There were also questions of how to test this infrastructure, which, Gustavo clarified, can be done using the sw_sync driver. Finally, folks asked about the status of the patches to libsync in AOSP, which were clarified to be in their 3rd revision and are getting quite close. Developers from Google commented that the next revision should be able to be merged. 

(See also: [coverage of Gustavo's XDC talk](<https://lwn.net/Articles/702339/>) on mainlining the sync framework.) 

#### More to come

As mentioned at the beginning, this microconference covered a large range of topics. The second part in this series, to appear shortly, will finish out our coverage of this gathering; stay tuned.
