---
title: "ELC: Linaro power management work"
url: https://lwn.net/Articles/439169/
date: "April 20, 2011"
category: Power management
author: "By Jake Edge April 20, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
April 20, 2011

There was a large Linaro presence at this year's [Embedded Linux Conference](<http://events.linuxfoundation.org/events/embedded-linux-conference/>) with speakers from the organization reporting on its efforts to consolidate functionality from the various ARM architecture trees. One of those talks was by Amit Kucheria, technical lead for the power management working group (PMWG), who talked about what the working group has been doing since it began. That includes some work on tools like powertop, and the newly available PowerDebug, as well as some consolidation within the kernel tree. He also highlighted areas where Linaro plans to focus its efforts in the future. 

Kucheria started with a look at what Linaro is trying to accomplish, part of which is to ""take the good things in the BSP [board support package] trees and get them upstream"". In addition, consolidating the kernel source, so that there is one kernel tree that can be used by all of the Linaro partners, is high on the list. There is a fair amount of architecture consolidation that is part of that, including things like reducing the ""ten or twenty `memcpy()` functions"" to one version optimized for all of the ARM processors. All of that work should result in patches that get sent upstream. 

The PMWG has ""existed for six to eight months now"", Kucheria said, and has been focused on consolidation and tools. There has been a bit of kernel work, which includes ensuring that the clock tree is exported in the right place in debugfs for five System-on-a-chips (SoCs) that Linaro and its sponsors/partners have targeted (Freescale i.MX51, TI OMAP 3 and 4, Samsung Orion, and ST-Ericsson UX8500). In addition, work was done on cpufreq, cpuidle, and CPU hotplug for some of them. Some of that work is still in progress, but most of it has gone (or is working its way) upstream, he said. 

[ ![\[PowerDebug\]](https://static.lwn.net/images/2011/elc-powerdebug-sm.png) ](<https://lwn.net/Articles/439350/>)

Beyond kernel work, the group has been working on tools, starting with getting powertop to work with ARM CPUs and pushing that work upstream. A new tool, [PowerDebug](<https://wiki.linaro.org/WorkingGroups/PowerManagement/Doc/PowerDebug>), has been created to help look at the clock tree to see ""what clocks are on, which are active, and at what frequency"", Kucheria said. It also shows power regulators that have registered with the regulator framework by pulling information from sysfs. It shows which regulators are on and what voltages are currently being used. Other SoCs or architectures can use PowerDebug simply by exporting their clock tree into debugfs. 

PMWG has also been [experimenting](<https://wiki.linaro.org/WorkingGroups/PowerManagement/Doc/Hotplug>) with thermal management and hotplug. In particular, it has been looking at what policies make sense when the CPU temperature gets too high. One possibility would be to hot-unplug a core to reduce the amount of heat generated. There is some inherent latency in plugging or unplugging a core, he said, which can range from 40-50ms in a simple case to several seconds if there are a lot of threads running. There is a notification chain that causes the latency, so it's possible that could be reduced by various means. 

#### Complexity in power management

[ ![\[Diagram\]](https://static.lwn.net/images/2011/elc-kucheria-diagram-sm.png) ](<https://lwn.net/Articles/439336/>)

With a slide showing the complexity of Linux power management (shown at right) today, Kucheria launched into a description of some of the problems that OEMs are faced with when trying to tune products for good battery life. In that diagram, he noted there are ""six or seven different knobs that you can twiddle"" to adjust power usage. Those OEMs simply don't have the resources to deal with that complexity, some kind of simplification is required. In addition, the complexity is growing with more and more SoCs along with different power management schemes in the hardware. 

In the "good old days", of five or six years ago, the OMAP 1 just used the Linux driver model suspend hooks to change the clock frequency. The clock framework was standard back then, but now there are 30 or 40 different clock frameworks in the ARM tree. CPU frequency scaling (cpufreq) was added after that, but it doesn't take into account the bus or coprocessor frequencies. Later on, several different frameworks were added including the regulator framework, cpuidle to control idle states, and power management quality of service (pm_qos). 

The quality of service controls are important for devices that need to bound the latency for coming out of idle states, for example for network drivers that cannot tolerate more than 300ms of latency. The cpuidle framework introduced some problems, though, Kucheria said, because they were created by Intel, who concentrated on its platforms. The C-states (C0-C6) don't really exist for ARM processors and various vendors interpreted them differently for particular SoCs. In addition, some have added additional states (C7, C8) 

Later still in the evolution of Linux power management, hotplug support was added, which can reduce the power consumption by unplugging CPU cores. There are a number of outstanding issues there, though, including latency and policy. Vendors have various ""patches floating around"", but there isn't a consistent approach. Coming up with policies, perhaps embodied in a hotplug governor, is something that needs to be done. 

Runtime power management was the next component added in. PMWG would like to use it to reduce the need for drivers to talk directly to the clocks and instead they would talk in a more general way to the runtime power management framework. Lots of code that is scattered around in various drivers can be centralized in bus drivers, which will make the device drivers much more portable because they don't refer to specific clocks. Vendors have started switching over to using the runtime power management framework, but ""it's a painful process"" to change all of the drivers, he said. 

The latest piece of the power management puzzle is the addition of Operating Performance Points (OPP) support, which was added in 2.6.38. OPP is a way to describe frequency/voltage pairs that a particular SoC will support for its various sub-modules. OPP is very CPU/SoC-specific, but can also encapsulate the requirements for different buses and co-processors. The cpufreq framework can make use of the information as it changes the frequency characteristics of different parts of the hardware. 

As more dual-core and quad-core packages are being used, heat can be a problem. The existing thermal management framework is not being used by ARM vendors yet and there are a number of issues to be resolved. Linaro wants to ""figure it out once and for all"", and that is one its focuses in the coming months. One of the questions is what should be done when the system is overheating. Should it unplug one or more cores? Or reduce the frequency of the CPU clock? One of the ""crazy things"" PMWG has been thinking about is registering devices that can reduce their frequency as "cooling devices" (since they will generate less heat with a lower frequency). 

#### PMWG's plans

The existing thermal management code works for desktop Linux, Ubuntu in particular, and also for Android, but there is still some experimenting that needs to be done to come up with an ARM-wide solution. Another area that PMWG will work on is adding scheduling domains for ARM so that you can ""tweak your scheduler policy"" regarding how processes and threads get spread around on multiple cores. Scheduling domains and sched_mc tunables could eliminate the need for hotplug in some cases, he said. 

Rationalizing the names and abilities of the processor C-states is also something that PMWG will be working on. Kucheria said that PMWG wants to ""start a conversation"" with the relevant vendors and developers to make that happen. PowerDebug enhancements are also on the radar: ""If you need stuff [in PowerDebug], let us know"". There is lots of other consolidation work that could be done, but there are only enough developers to address the parts he described, at least in the near term. 

At the end of the talk, Kucheria put the Linux power management diagram slide back up, noting that the complexity was ""great for job security"". There is clearly plenty of work to do in the ARM tree in the months ahead. Kucheria's talk just covered the work going on in the power management group, but there are four other groups within Linaro (kernel, toolchain, graphics, and multimedia) that are doing similar jobs inside and outside of the kernel. One gets the sense that the companies who founded Linaro were getting as tired of the chaotic ARM world as the kernel developers (e.g. Linus Torvalds) are. So far, the organization has made some strides, but there is a long way to go.
