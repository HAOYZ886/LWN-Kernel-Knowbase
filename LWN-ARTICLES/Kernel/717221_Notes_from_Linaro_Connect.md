---
title: Notes from Linaro Connect
url: https://lwn.net/Articles/717221/
date: "March 15, 2017"
category: "Development tools-Testing; Scheduler-and power management"
author: "By Jonathan Corbet March 15, 2017 Linaro Connect"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 15, 2017

* * *

[Linaro Connect](<https://lwn.net/Archives/ConferenceByYear/#2017-Linaro_Connect>)

The first of two 2017 Linaro Connect events was held March 6 to 10 in Budapest, Hungary; your editor had the privilege of attending. Reports from a number of the sessions there have appeared in separate articles. There were a number of discussions at the event that, while not being enough to fill an article on their own, were nevertheless worthy of some attention. 

Connect is an interesting event, in that it is a combination of an architecture-specific kernel developers' gathering and a members-only meeting session. Not being a member, your editor only participated in the former aspect. Sessions at Connect are usually short — 25 minutes — and focused on a specific topic; they also routinely run over their allotted time. There is an emphasis on discussion, especially in the relatively unstructured "hack sessions" that occupy much of the schedule. Many of the sessions are focused on training: how to upstream code, for example, or kernel debugging stories in Mandarin ([video](<http://connect.linaro.org/resource/bud17/bud17-tr07/>)). 

A lack of Mandarin language skills will prevent coverage of that last session, but there were others that were somewhat more comprehensible to the Mandarin-impaired. 

#### Kernelci.org

Mark Brown, Tyler Baker, and Matt Hart ran a session about [kernelci.org](<https://kernelci.org/>), arguably one of the community's most underappreciated testing resources. This operation, run by Linaro and BayLibre, performs automatic build-and-boot testing for the kernel; its infrastructure is hosted by Linaro and Collabora. Tests are run on the ARM, ARM64, MIPS, and x86 architectures. For every commit that is made, tests are run with every in-tree `defconfig` file — over 260 builds for every commit. The resulting kernels are booted on over 100 different boards. This operation, Brown said, greatly increases the likelihood that kernels will build and, as a result, the number of failed configurations is going down over time. That, in turn, makes merge windows less stressful. 

As Baker explained, this testing structure is driven by the [LAVA](<https://www.linaro.org/initiatives/lava/>) tool; it serves as a scheduler and job runner for board farms. LAVA is [used and distributed by Debian](<https://wiki.debian.org/LAVA>), he said. 

LAVA is completing a big transition to the v2 release, which makes a number of significant changes, Hart said. Job files are now created with [Jinja2](<http://jinja.pocoo.org/docs/2.9/>) templates, an improvement over the hand-written JSON used in v1. Jobs are run asynchronously, without polling, and [ZeroMQ](<http://zeromq.org/>) is used for communications. [ReactOBus](<https://github.com/ivoire/ReactOBus>) is used to run jobs from messages. LAVA v1 tried to apply a fair amount of magic to hide the differences between different test systems, but that proved hard to work with. So v2 requires more explicit configuration in this area. 

The v2 system is settling in, but a permanent home for the ReactOBus daemon is yet to be identified. [[Video](<http://connect.linaro.org/resource/bud17/bud17-105/>)] 

#### Load tracking

Vincent Guittot ran a session on load tracking — keeping track of how much load each process puts on the CPU. Accurate load tracking can help the scheduler make better decisions about task placement in the system; it can also be helpful when trying to minimize the system's power consumption. The [per-entity load tracking](<https://lwn.net/Articles/531853/>) (PELT) mechanism in the kernel is better than what came before, but it is proving to not work as well as desired, especially when it comes to power management. The window-assisted load tracking (WALT) patches (described in [this article](<https://lwn.net/Articles/706374/>)) improve the situation, but that work has not made it into the mainline. 

The complaints with PELT are that it is not responsive enough and that its metrics are not always stable. The tracking of small tasks can be inaccurate, causing a mostly idle CPU to appear to be busy. Load is not propagated between control groups when a task is migrated which, among other things, can cause erroneous CPU-frequency changes. The good news is that around 20 patches improving PELT have been merged since 4.7; they fix the small-task tracking and load-propagation issues. Upcoming work should improve the handling of blocked loads and address some of the frequency scaling issues. 

A related problem is that the community has lacked realistic benchmarks to measure the results of load-tracking changes; that is being addressed with new tests. There are some interesting interactions with the processor's thermal management mechanisms, though. When asked, Guittot said that there has not been a lot of power-consumption testing so far; most of the work to this point has been focused on performance. 

Future work includes improving utilization tracking for realtime tasks, which are currently not part of the load-tracking mechanism. There are also some practical problems on current hardware. Realtime tasks want to run at the maximum frequency, but a frequency change on a HiKey board takes 1.5ms. A realtime task needing 2ms of run time will not get maximum-frequency performance. A more responsive load-tracking mechanism could help the scheduler ensure that the CPU is running at the needed speed. There is also a focus on improving responsiveness, which comes down to ensuring that the CPU frequency is increased quickly when the need arises. A slow ramp-up will lead to observable behaviors like jumpy scrolling. Finally, there is a desire to improve the responsiveness of the PELT system, perhaps by introducing the windowing technique used in WALT. [[Slides](<http://connect.linaro.org/resource/bud17/bud17-218/>)] 

#### ARM Mali drivers

Many ARM systems come equipped with the ARM Mali GPU. In a session on the state of the Mali drivers, John Reitan and Ketil Johnsen described some of the work that is being done in this area. There is a [software development kit](<https://github.com/ARM-software/vulkan-sdk>) for the [Vulkan](<https://en.wikipedia.org/wiki/Vulkan_\(API\)>) graphics API available under the BSD license. Not all GPUs support Vulkan, though; in particular, the HiKey board used by many ARM developers has no Vulkan support. Within a month or so ARM should be releasing its compute library that allows running code on the GPU. It may be useful for image-recognition tasks and more. It will show up on the [ARM GitHub page](<https://github.com/ARM-software/>) once it's ready. 

With the news items out of the way, the audience quickly moved the discussion to the topic its members were really interested in: prospects for open-sourcing the Mali driver code. The answer was that ARM has no intention of doing so, mostly out of fear of unspecified "patent issues". The risk of patent trolls is simply too great to allow that release to happen. This was not, of course, the answer that the audience wanted to hear, but nobody was particularly surprised. 

Arnd Bergmann suggested that perhaps a free Vulkan driver could be released; Vulkan is simpler than the full OpenGL API and might thus pose a lower risk. The speakers are not lawyers and could not respond to that suggestion beyond agreeing that it is worth considering. Meanwhile, there is a possibility that free drivers for some subcomponents could be released in the relatively near future. 

A related pain point around Mali is the lack of device-tree bindings in the kernel. The normal rule is that bindings are only accepted for drivers that are, themselves, in the kernel; there is no Mali driver there, thus no bindings. But that has led every SoC vendor to come up with its own customized bindings. There has been talk of loosening the rules a bit to allow the addition of bindings for some out-of-tree drivers to reduce this pain. 

John Stultz pointed out that running the Mali drivers on mainline kernels is often difficult, and wondered if there were any improvements expected in that area. Development effort on the binary-only driver tends to be focused on kernels the customers are using, and those kernels are usually old. Internally, the Mali driver does usually work on the mainline, but it can take months for the patches to get out to the rest of the world. 

It's also hard for distributors who would like to make the binary-only driver available to their users. One recent improvement, at least, is that the license on that driver has changed to allow it to be distributed. But it is still difficult to make a package that works on even a subset of boards. Meanwhile, every driver release tends to break systems, and the driver tends to break with kernel updates. As Grant Likely pointed out, having to keep the kernel and the user-space driver code in lockstep makes the creation of any sort of generic distribution difficult. It was agreed that a better job needs to be done here. [[Video](<http://connect.linaro.org/resource/bud17/bud17-421/>)] 

For those interested in other Connect sessions, the full set of videos and slides from the "BUD17" event can be found on [this page](<http://connect.linaro.org/bud17/resources/>). The next Connect will be held September 25 to 29 in San Francisco, California. 

[Thanks to Linaro and the Linux Foundation for funding your editor's travel to Connect.]
