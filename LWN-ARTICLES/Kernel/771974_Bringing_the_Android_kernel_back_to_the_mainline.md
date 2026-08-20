---
title: Bringing the Android kernel back to the mainline
url: https://lwn.net/Articles/771974/
date: "November 15, 2018"
category: "Android-Generic kernel image"
author: "By Jonathan Corbet November 15, 2018 Linux Plumbers Conference"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 15, 2018

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2018-Linux_Plumbers_Conference>)

Android devices are based on the Linux kernel but, since the beginning, those devices have not run _mainline_ kernels. The amount of out-of-tree code shipped on those devices has been seen as a problem for most of this time, and significant resources have been dedicated to reducing it. At the 2018 [Linux Plumbers Conference](<http://linuxplumbersconf.org/>), Sandeep Patil talked about this problem and what is being done to address it. The dream of running mainline kernels on Android devices has not yet been achieved, but it may be closer than many people think. 

Android kernels, he said, start their life as a long-term stable (LTS) release from the mainline; those releases are combined with core Android-specific code to make the [Android Common Kernel](<https://source.android.com/devices/architecture/kernel/android-common>) releases. Vendors will pick a common kernel and add a bunch more out-of-tree code to create a kernel specific to a system-on-chip (SoC) and ship that to device manufacturers. Eventually one of those SoC kernels is frozen, perhaps with yet another pile of out-of-tree code tossed in, and used as the kernel for a specific device model. It now only takes a few weeks to merge an LTS release into the Android Common Kernel, but it's still a couple of years before that kernel shows up as a device kernel. That is why Android devices are always running ancient kernels. 

There are a lot of problems associated with this process. The Android core has to be prepared to run on a range of older kernels, a constraint that makes it hard to use newer kernel features. Kernel updates are slow or, [![\[SandeepPatil\]](https://static.lwn.net/images/conf/2018/lpc/SandeepPatil-sm.jpg)](<https://lwn.net/Articles/771983/>) more often, nonexistent. The use of large amounts of out-of-tree code (as in millions of lines of it) makes it hard to merge in new stable updates, and even when that's possible, shipping the result to users is frightening to vendors and not often done. There is no continuous-integration process for Android kernels, and it's not possible to run Android systems on mainline kernels. All told, the way Android kernels are developed and managed takes away a lot of the advantages of using Linux in the first place, but work is being done to address many of these issues. 

With regard to older kernels: the Oreo release required the use of one of the 3.18, 4.4, or 4.9 kernels — an improvement over previous releases, which had no kernel-version requirements at all. The Pie release narrowed the requirements further, saying that devices must ship with 4.4.107, 4.9.84, or 4.14.42 (or a later stable release, in each case). The Android developers are trying to "push things up a notch" by mandating the incorporation of stable updates. This has improved the situation, but the base kernel remains two years old (or more), and the Android core still has to work on kernels back to 3.18. 

Patil noted that some people worry about regressions from the stable updates, but in two years of incorporating those stable updates, the Android project has only encountered one regression. In particular, 4.4.108 broke things, which is why nothing later than 4.4.107 is required at the moment. Otherwise, he said, the stable updates have proved to be highly reliable for Android systems. 

One reason for that may be that the situation with continuous-integration testing is improving; the [LKFT effort](<https://lkft.linaro.org/tests/>) is now running functional testing on the LTS, ‑rc, and Android Common kernels, for example. More testing is happening through [KernelCI](<https://kernelci.org/>), and Android developers are contributing to the [Linux Test Project](<http://linux-test-project.github.io/>) as well. Kernel patches go through pre-submission testing on an emulated device called [Cuttlefish](<https://github.com/google/android-cuttlefish>), which can run both Android and mainline kernels. More testing is being done by SoC vendors, none of whom have reported problems from LTS kernel updates so far. They do see merge conflicts with their out-of-tree code, but that is unsurprising. 

Even so, kernel upgrades remain a huge issue for Android vendors, who worry about shipping large numbers of changes to deployed devices. So devices generally don't get upgraded kernels after they ship — a bad situation, but it's better than the recent past, when kernels could not be upgraded for a given SoC after its launch, he said. Google plans to continue to push vendors to ship updates, though, eventually mandating updates to newer LTS releases even after a device is launched. At some point, LTS releases will be included in Android security bulletins, because there really is value in getting all of the bug fixes. Patil echoed Greg Kroah-Hartman's statement that there are no "security bugs" as such; "there are just bugs" and they should all be fixed. 

The problem of devices being unable to run mainline kernels remains; the problem, of course, is all of that out-of-tree code. The amount of that code in the Android Common Kernel has been reduced considerably, though, with a focused effort at getting the changes upstream. There are now only about 30 patches in the Android Common Kernel, adding about 6,500 lines of code, that are needed to boot Android. The eventual plan is to push that to zero, but there are a number of issues to deal with still, including solving problems with priority inheritance in binder, getting energy-aware scheduling into the mainline, and upstreaming the SDCardFS filesystem bridge. 

[Project Treble](<https://lwn.net/Articles/765467/>), he said, introduced a new "vendor interface" API that implements a sort of hardware abstraction layer. Along with this interface came the concept of a [generic system image](<https://source.android.com/setup/build/gsi>) (GSI), being a build of the Android Open Source Project that can be booted on any Android device. If the GSI can be booted on a specific device, then the manufacturer has implemented the vendor interface correctly. 

For now, the kernel is considered to be part of the vendor interface — the vendor must provide it as part of the low-level implementation. The plan, though, is for Android to provide a generic kernel image based on the mainline. Devices will be expected to run this kernel; to make that happen, vendors will provide a set of kernel modules to add the necessary hardware support. Getting there will require the upstreaming of [kernel symbol namespaces](<https://lwn.net/Articles/760045/>) among other things. 

This design will clearly not eliminate the out-of-tree code problem, since those modules will, in many or most cases, not come from the mainline. But there is still a significant change here: vendor-specific code will be relegated to loadable modules and, thus, be unable to change the core kernel. The days of vendors shipping their own CPU schedulers should come to an end, for example; all out-of-tree code will have to work with the generic kernel image using the normal module interface. That will force that code into a more upstream-ready state, which is a step in the right direction. 

In conclusion, Patil said, the Android kernel team is now aggressively trying to upstream code before shipping it. There is a renewed effort to proactively report vulnerabilities and other problems and to work with upstream to resolve them. Beyond the above, the project has a number of goals, including getting the ashmem and ion modules out of the staging tree, improving Android's use of device trees, and more. But things are progressing; someday, the "Android problem" may be far behind us. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to LPC.]
