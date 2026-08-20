---
title: Creating a kernel build farm
url: https://lwn.net/Articles/702375/
date: "October 5, 2016"
category: Build system
author: "By Jonathan Corbet October 5, 2016 Kernel Recipes"
---

By **Jonathan Corbet**  
October 5, 2016

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2016-Kernel_Recipes>)

Willy Tarreau, known for his maintenance of extra-long-term stable kernels among other things, is not often seen at Linux events. Attendees at Kernel Recipes 2016 thus got a rare opportunity when Willy discussed his work on the creation of build farms for the kernel (or any other large software project). Not every developer needs a build farm, but those who do a lot of backports, must frequently bisect to find problems, have slow laptops, or maintain a large body of code will find that a dedicated build infrastructure can save a lot of time. 

Maintaining a stable kernel can require doing a lot of backports, which is not trivial work. There are API changes to contend with, and backported patches can often cause build or boot failures. Thus, every backported patch must be independently validated. This process takes a lot of time in general, and kernel build time tends to dominate the rest. It takes 45 minutes to do an allmodconfig build on his laptop, for example. So he will often start a build and go for lunch — only to discover that the build failed shortly after he left and must be restarted from the beginning. 

Developers like Willy thus clearly have an interest in reducing the amount of time they spend waiting for builds. One way to do that would to stop testing backports but, he allowed, that may not be the best of ideas. He could simply buy a bigger machine but, beyond the financial issues, a large build machine is not something that he can carry around with him. One can use tools like [ccache](<https://ccache.samba.org/>), but it tends to not work well when include files are changed, which is a common occurrence in backporting work. An option that does have promise, though, is distributing the build work across multiple machines. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Software issues

Not every task is suitable for a build farm, he said; one has to start with a workload that is readily distributable. So the project's build system must support parallel builds; a surprising number of projects don't support that. The project should be large, with many more source files than machines to build them on, and the compile time for each file should be approximately equal. The kernel, as it happens, fits this pattern well. 

Needless to say, a developer needs to have multiple machines available to run the build farm on. Importantly, each machine needs to be running the exact same compiler, or problems will result later on. The only reliable way to get there, he said, is to build the compiler yourself. It is a scary process, but it's actually not that hard. [Crosstool-NG](<http://crosstool-ng.org/>) is good for this task; it can also do [![\[Willy Tarreau\]](https://static.lwn.net/images/conf/2016/recipes/WillyTarreau2-sm.jpg)](<https://lwn.net/Articles/702380/>) "[Canadian cross-compiler builds](<https://en.wikipedia.org/wiki/Cross_compiler#Canadian_Cross>)" (using an x86 machine to create an ARM-native compiler that does x86 compiles, for example) that can be useful for some kinds of build farms. 

Also needed is a way to submit jobs to machines — a distributed build controller. This system needs to be unintrusive, not requiring patches to the system being built; it should also have little overhead "or you lose." A good build controller can work with cross compilers, and it should be able to fall back to the local node when dependency issues get in the way of remote builds. The ability to work around unreachable machines is important; machines must be able to drop out of the build farm without stopping everything. 

The right tool for this job is [distcc](<https://github.com/distcc>). It can work as a wrapper, or in "masqueraded mode," where distcc uses the name by which it was invoked to find the real compiler on the remote systems. It requires no daemons on the build systems, can implement per-node usage limits, is able to avoid using unresponsive machines, and more. There are a few stumbling blocks when one uses distcc with the kernel; it will not use the remote machines if `gcov` code coverage tracking is enabled, for example, so one must ensure that `CONFIG_GCOV_KERNEL` is turned off. One should also remember that the preprocessing and final linking steps are done on the local node; they take 20-30% of the total time and, as a result, place a lower bound on how fast the build can ever be. 

#### The hardware side

Software is important for the creation of a build farm but, obviously, one will not get far without hardware to build on. Much of the rest of the talk was dedicated to the process of picking machines for the farm. To start, he said, it's important to know what is being optimized for. Some developers want the highest performance they can get for a given cost; others will want to minimize the number of nodes, the power consumption, or the noise created by the farm. Different needs will lead to different choices. 

When comparing machine performance, Willy said, it is important to always use the same project for build testing. In the end, the metric that matters is lines compiled per second, so that is what should be measured. Make sure that the CPU frequency governor is set for performance ("make the planet warmer," he said) as the powersave mode often used on laptops can slow the build considerably. One should make multiple runs, looking for at least three with consistent results. 

What affects the performance of a build node? The CPU architecture and frequency matter, of course; the recent Intel and AMD processors, he said, are impressive. Memory latency matters, as do the size and latency of the CPU caches. Storage speed will have an effect on performance. The options used to build the compiler can also have up to a 10% effect on performance; [![\[Willy Tarreau\]](https://static.lwn.net/images/conf/2016/recipes/WillyTarreau-sm.jpg)](<https://lwn.net/Articles/702381/>) playing with crosstool-NG to get the best configuration is recommended. In the end, he has found DRAM latency to be the single most important factor affecting a machine's build performance, followed by cache latency, CPU frequency, and core counts, in that order. If the system is held back by its DRAM performance, adding CPUs will not help the situation. 

To get the best performance, one should ensure that the build systems run at 100% CPU utilization — except the local node, which should not be heavily loaded. Distributed builds create a lot of network traffic; the network must not be saturated or things will slow down. All of the memory channels should be used; a PC with a single memory stick in it will not perform well. Enabling hyperthreading can give a 50% performance gain, and overclocking can be worth looking into, depending on the system involved. There is no point in having more than eight cores, there won't be enough memory bandwidth to keep them busy. In general, he said, machines designed for gamers are the best; they are not hugely expensive and are built to be tweakable. 

If one is optimizing for a low number of nodes, one obviously wants the highest performance possible per node. Willy suggests using dual-socket, eight-core machines with all memory slots populated. Such a machine will have a huge memory bandwidth and will perform well. It will also be big, power-hungry, and noisy. 

#### Going small

One could, instead, optimize for hardware cost. A [NanoPi NEO](<http://www.friendlyarm.com/index.php?route=product/product&product_id=132>) system can be had for $8; it is a quad-core, 1.2 GHz machine. It actually works, though it is about 1/16 the power of a PC, and gets even slower when thermal throttling kicks in. This machine is limited by its 100MB/sec Ethernet, though. By the time one figures in the additional costs — shipping, switch ports, network cables, SD cards, USB power supplies, etc. — the cost starts to go up. As a whole, this approach is not entirely interesting. 

One can do a bit better with mid-range machines like the [NanoPi 2 Fire](<http://nanopi.io/nanopi2-fire.html>) or the [ODROID C2](<http://www.hardkernel.com/main/products/prdt_info.php>). These machines run $25-60; they offer good performance density and reasonable cooling. On the other hand, the per-board cost is high, some can run hot, and some of them require special kernels to operate. 

At the high end (of cheaper hardware), there are devices like the [MiQi](<https://forum.mqmaker.com/t/announcing-miqi-a-credit-card-sized-computer/371>), the RKM v5, or the CS 008\. These are often sold as "set-top boxes" in the $50-200 range. They can offer 4-8 ARM A17 or A53 cores, making them relatively powerful boards with 1/4 the performance of a basic PC. Some of them offer gigabit Ethernet and onboard storage; some even have mainline kernel support. On the other hand, they often are sold running Android, and can be of varying build quality. Power consumption can be higher than advertised. They are subject to thermal throttling; he said to never buy such a board if it does not have a heat sink installed. The MiQi has become his favorite in this area. 

To prove the point, Willy pulled out his MiQi-based build farm, consisting of four MiQi boards, a small Ethernet switch, and a USB power supply, from a small box in his backpack, and proceeded to build a kernel with it. The process takes less than 15 minutes "when it works." 

He concluded by mentioning a few "future research" ideas, starting with playing with the [distcc pump mode](<https://opensource.googleblog.com/2008/08/distccs-pump-mode-new-design-for.html>). He would like to experiment with putting [HAProxy](<http://www.haproxy.org/>) in front of distcc; in his experience, the loads are not always well balanced across the build machines now. Build farms could be smaller and cheaper if they could be built without an Ethernet switch. WiFi is not up to the traffic demands and cannot be used, though; he wants to try USB networking instead. 

As the talk wound down, an audience member asked about distribution recommendations. Willy responded that pretty much any distribution can be made to work in this role. In fact, the Android-based boards can even support this work, but only if the SELinux configuration is not so restrictive that it gets in the way. 

[Your editor thanks Kernel Recipes for supporting his travel to this event.]
