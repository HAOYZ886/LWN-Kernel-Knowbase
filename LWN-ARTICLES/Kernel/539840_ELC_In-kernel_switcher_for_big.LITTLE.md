---
title: "ELC: In-kernel switcher for big.LITTLE"
url: https://lwn.net/Articles/539840/
date: "February 27, 2013"
category: "Architectures-Arm; big.LITTLE"
author: "By Jake Edge February 27, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
February 27, 2013

The ARM big.LITTLE architecture has been the subject of a number of [LWN articles](<https://lwn.net/Articles/481055/>) (here's [another](<https://lwn.net/Articles/501501/>)) and [conference talks](<https://lwn.net/Articles/514063/>), as well as a fair amount of code. A number of upcoming systems-on-chip (SoCs) will be using the architecture, so some kind of near-term solution for Linux support is needed. Linaro's Mathieu Poirier came to the 2013 [Embedded Linux Conference](<http://events.linuxfoundation.org/events/embedded-linux-conference/>) to describe that interim solution: the in-kernel switcher. 

#### Two kinds of CPUs

Big.LITTLE incorporates architecturally similar CPUs that have different power and performance characteristics. The similarity must consist of a one-to-one mapping between instruction sets on the two CPUs, so that code can "migrate seamlessly", Poirier said. Identical CPUs are grouped into clusters.

[ ![\[Mathieu Poirier\]](https://static.lwn.net/images/2013/elc-poirier-sm.jpg) ](<https://lwn.net/Articles/540195/>)

The SoC he has been using for testing consists of three Cortex-A7 CPUs (LITTLE: less performance, less power consumption) in one cluster and two Cortex-A15s (big) in the other. The SoC was deliberately chosen to have a different number of processors in the clusters as a kind of worst case to catch any problems that might arise from the asymmetry. Normally, one would want the same number of processors in each cluster, he said. 

The clusters are connected with a cache-coherent interconnect, which can snoop the cache to keep it coherent between clusters. There is an interrupt controller on the SoC that can route any interrupt from or to any CPU. In addition, there is support in the SoC for I/O coherency that can be used to keep GPUs or other external processors cache-coherent, but that isn't needed for Linaro's tests. 

The idea behind big.LITTLE is to provide a balance between power consumption and performance. The first idea was to run CPU-hungry tasks on the A15s, and less hungry tasks on the A7s. Unfortunately, it is "hard to predict the future", Poirier said, which made it difficult to make the right decisions because there is no way to know what tasks are CPU intensive ahead of time. 

#### Two big.LITTLE approaches

That led Linaro to a two-pronged approach to solving the problem: Heterogeneous Multi-Processing (HMP) and the In-Kernel Switcher (IKS). The two projects are running in parallel and are both in the same kernel tree. Not only that, but you can enable either on the kernel command line or switch at run time via sysfs. 

With HMP, all of the cores in the SoC can be used at the same time, but the scheduler needs to be aware of the capabilities of the different processors to make its decisions. It will lead to higher peak performance for some workloads, Poirier said. HMP is being developed in the open, and anyone can participate, which means it will take somewhat longer before it is ready, he said. 

IKS is meant to provide a "solution for now", he said, one that can be used to build products with. The basic idea is that one A7 and one A15 are coupled into a single virtual CPU. Each virtual CPU in the system will then have the same capabilities, thus isolating the core kernel from the asymmetry of big.LITTLE. That means much less code needs to change. 

Only one of the two processors in a virtual CPU is active at any given time, so the decision on which of the two to use can be made at the CPU frequency (cpufreq) driver level. IKS was released to Linaro members in December 2012, and is "providing pretty good results", Poirier said. 

An alternate way to group the processors would be to put all the A15s together and all the A7s into another group. That turned out to be too coarse as it was "all or nothing" in terms of power and performance. There was also a longer synchronization period needed when switching between those groups. Instead, it made more sense to integrate "vertically", pairing A7s with A15s. 

For the test SoC, the "extra" A7 was powered off, leaving two virtual CPUs to use. The processors are numbered (A15_0, A15_1, A7_0, A7_1) and then paired up (i.e. {A15_0, A7_0}) into virtual CPUs; "it's not rocket science", Poirier said. One processor in each group is turned off, but only the cpufreq driver and the switching logic need to know that there are more physical processors than virtual processors. 

The virtual CPU presents a list of operating frequencies that encompass the range of frequencies that both A7 and A15 can operate at. While the numbers look like frequencies (ranging from 175MHz to 1200MHz in the example he gave), they don't really need to be as they are essentially just indexes into a table in the cpufreq driver. The driver maps those values to a real operating point for one of the two processors. 

#### Switching CPUs

The cpufreq core is not aware of the big.LITTLE architecture, so the driver does a good bit of work, Poirier said, but the code for making the switching decision is simple. If the requested frequency can't be supported by the current processor, switch to the other. That part is eight lines of code, he said. 

For example, if virtual CPU 0 is running on the A7 at 200MHz and a request comes in to go to 1.2GHz, the driver recognizes that the A7 cannot support that. In that case, it decides to power down the A7 (which is called the outbound processor) and power up the A15 (inbound). There is a synchronization process that happens as part of the transition so that the inbound processor can use the existing cache. That process is described in Poirier's [slides [PDF]](<http://events.linuxfoundation.org/images/stories/slides/elc2013_poirier.pdf>), starting at slide 17. 

The outbound processor powers up the inbound and continues executing normal kernel/user-space code until it receives the "inbound alive" signal. After sending that signal, the inbound processor initializes both the cluster and interconnect if it is the first in its cluster (i.e. the other processor of the same type, in the other virtual CPU is powered down). It then waits for a signal from the outbound processor. 

Once the outbound processor receives "inbound alive" signal, the blackout period (i.e. time when no kernel or user code is running on the virtual CPU) begins. The outbound processor disables interrupts, migrates the interrupt signals to the inbound processor, then saves the current CPU context. Once that's done, it signals the inbound processor, which restores the context, enables interrupts, and continues executing from where the outbound processor left off. All of that is possible because the instruction sets of the two processors are identical. 

As part of its cleanup, the outbound processor creates a new stack for itself so that it won't interfere with the inbound. It then flushes the local cache and checks to see if it is the last one standing in its cluster; if so, it flushes the cluster cache and disables the cache-coherent interconnect. It then powers itself off. 

There are some pieces missing from the picture that he painted, Poirier said, including ["vlocks" and other mutual exclusion mechanisms](<https://lwn.net/Articles/539082/>) to handle simultaneous desired cluster power states. Also missing was discussion of the "early poke" mechanism as well as code needed to track the CPU and cluster states. 

#### Performance

One of Linaro's main targets is Android, so it used the interactive power governor for its testing. Any governor will work, he said, but will need to be tweaked. A second threshold (`hispeed_freq2`) was added to the interactive governor to delay going into "overdrive" on the A15 too quickly as those are "very power hungry" states. 

For testing, BBench was used. It gives a performance score based on how fast web pages are loaded. That was run with audio playing in the background. The goal was to get 90% of the performance of two A15s, while using 60% of the power, which was achieved. Different governor parameters gave 95% performance with 65% of the power consumption. 

It is important to note that tuning is definitely required--without it you can do worse than the performance of two A7s. "If you don't tune, all efforts are wasted", Poirier said. The interactive governor has 15-20 variables, but Linaro mainly concentrated on `hispeed_load` and `hispeed_freq` (and the corresponding `*2` parameters added for handling overdrive). The basic configuration had the virtual CPU run on the A7 until the load reached 85%, when it would switch to the first six (i.e. non-overdrive) frequencies on the A15. After 95% load, it would use the two overdrive frequencies. 

The upstreaming process has started, with the cluster power management code getting "positive remarks" on the ARM Linux mailing list. The goal is to upstream the code entirely, though some parts of it are only available to Linaro members at the moment. The missing source will be made public once a member ships a product using IKS. But, IKS is "just a stepping stone", Poirier said, and "HMP will blow this out of the water". It may take a while before HMP is ready, though, so IKS will be available in the meantime. 

[ I would like to thank the Linux Foundation for travel assistance to attend ELC. ]
