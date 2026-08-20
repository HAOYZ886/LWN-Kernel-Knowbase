---
title: The Sound Open Firmware project launches
url: https://lwn.net/Articles/749888/
date: "March 21, 2018"
category: Firmware
author: "By Jonathan Corbet March 21, 2018 ELC"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 21, 2018

* * *

[ELC](<https://lwn.net/Archives/ConferenceByYear/#2018-Embedded_Linux_Conference>)

It is an increasingly poorly kept secret that, underneath the hood of the components that most of us view as "hardware", there is a great deal of proprietary software. This code, written by anonymous developers, rarely sees the light of day; as a result, it tends to have all of the pathologies associated with software that nobody can either review or fix. The 2018 Embedded Linux Conference saw an announcement for a new project that, with luck, will change that situation, at least for one variety of hardware: audio devices. 

Intel's Imad Sousou took to the stage at ELC to introduce a couple of new projects. One of them was inspired by the [MinnowBoard](<https://minnowboard.org/>) project, which is working to create development boards based entirely on free software. That goal [![\[Imad Sousou\]](https://static.lwn.net/images/conf/2018/elc/ImadSousou-sm.jpg)](<https://lwn.net/Articles/749890/>) has been almost completely achieved, he said, but there is an exception in that the board requires a number of proprietary firmware components. The desire to fix that problem led to the creation of the [Sound Open Firmware](<https://www.sofproject.org/>) (SOF) project. 

The objective behind SOF is to create open-source firmware for audio and video-related devices. This is, Sousou said, an area that has typically been dominated by proprietary software. This project has been jointly launched with Google; there are, he suggested, plenty of opportunities for contributors who might want to join this effort. The code is released under a combination of the BSD and GPLv2 licenses. 

Sousou's talk stopped there, but a bit of digging can turn up some more information. The project has [a page on the ALSA project's wiki](<https://www.alsa-project.org/main/index.php/Firmware>) describing where to find the software and how to get started with it. There is a software-development kit (SDK) to install; building the firmware also requires a cross-compiler for the [Xtensa](<http://wiki.linux-xtensa.org/index.php/Main_Page>) architecture. The SDK includes an emulator that can be used during the development process; the SOF web site also notes that ""proprietary compilers and emulators"" are available. Currently, only Xtensa-based digital signal processors are supported, but the project's intent is to eventually support a wider range of hardware. 

The firmware itself is based on its own miniature kernel; it would appear that a new kernel was developed rather than adapting one of the (many) existing tiny kernels in circulation. It includes an earliest-deadline first scheduler and supports basic concepts like memory allocation, interrupt handling, work queues, and more. Using that, the firmware runs an "audio task" whose job is to move data between the DMA buffers and the audio-processing components. 

On the kernel side, SOF appears to present a semi-standard interface that all drivers can use, but there is still a significant amount of device-specific code required. The code to support SOF-based devices can be found in the ALSA system-on-chip tree (git://git.kernel.org/pub/scm/linux/kernel/git/lrg/asoc.git). It is not currently in the mainline; it also does not appear in linux-next, so it will almost certainly not make an appearance before the 4.18 development cycle. There is [a mailing list](<http://alsa-project.org/mailman/listinfo/sound-open-firmware>) with a fair amount of traffic; almost all of it comes from Intel addresses, which is unsurprising for a project that has only just been opened up to the world. 

This project is in its infancy (despite the nearly 18 months of history on the mailing list), but it has the potential to make things better in a number of ways. Manufacturers that participate in it may end up with higher-quality firmware with less effort. Developers interested in doing new things with audio hardware should find SOF-based devices to be a good starting point. And, if the project succeeds, we'll all have systems running a bit more free software at the lowest levels, which seems like a good thing. 

(For the curious, the other project announced in this talk was the [ACRN hypervisor](<https://projectacrn.org>). It is designed for safety-critical systems and appears to support many of the features found in systems like [seL4](<https://lwn.net/Articles/745946/>).) 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting your editor's travel to ELC.]
