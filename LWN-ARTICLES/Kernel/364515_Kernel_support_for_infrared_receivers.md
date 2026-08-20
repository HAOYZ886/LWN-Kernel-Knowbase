---
title: Kernel support for infrared receivers
url: https://lwn.net/Articles/364515/
date: "December 2, 2009"
category: "Development model-Driver merging; Device drivers"
author: "By Jonathan Corbet December 2, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 2, 2009

One of the stated goals of the staging tree is to bring widely-used drivers into the mainline kernel tree. This effort has been quite successful; the number of out-of-tree drivers has dropped considerably over the last year or so. There is one high-profile holdout, though: the [Linux Infrared Remote Control](<http://www.lirc.org/>) (LIRC) subsystem. LIRC is used to obtain input events from remote control devices and feed them through to applications; Linux-based digital video recorder systems are heavy LIRC users, but there are others as well. Back in October, Jarod Wilson [posted a new version of LIRC](<http://lwn.net/Articles/357828/>) for consideration. One month later, the kernel developers have started talking about it; what they lack in punctuality has been more than made up for in volume. 

One might think that merging this longstanding, heavily-used project into the mainline would not require a great deal of discussion. The problem is that LIRC brings with it a new ABI. Since user-space interfaces must be supported indefinitely, they tend to come under a higher degree of scrutiny than other parts of the code. LIRC has never had to freeze its ABI during its many years of out-of-tree existence, a freedom which has [made life easier](<https://lwn.net/Articles/364542/>) for its developers. But LIRC in mainline would not have this freedom, so any incompatible ABI changes need to be made prior to merging. And, as it happens, some developers would like to see significant changes. 

One would think that an IR receiver would be a simple device; all it must do is report button press and release events, much like a keyboard. Often, it seems, the simplest devices are the most complex to deal with. Some receivers have decoders built into them, allowing them to pass scan codes to the driver, which can then map them onto key codes to pass to applications. But others are simple indeed - they simply report the timing and length of pulses received from the remote. In this case, the driver must filter out glitches and perform protocol processing to get to the point where it can generate scan codes. For extra fun, there are a number of protocols in use, and some manufacturers have wisely decided that life would be much more interesting if they were to make their own versions of the protocols which differ from everybody else's. So the protocol processing can be painful and unpleasant. 

LIRC handles this mess by having drivers report "raw" pulse-length information via a special device; a user-space daemon then handles the task of turning that information into something that usefully describes a button-press event. In many cases, the low-level driver runs in user space and does not involve the kernel at all. Distribution of these events is also handled by the LIRC daemon, which can direct specific events to different applications, run programs in response to events, and so on in a flexible, scriptable manner. LIRC works, and some developers would like to see it merged into the mainline more-or-less as it stands now. Others, though, dislike the special-purpose "raw" interface used by LIRC. As Jon Smirl [put it](<https://lwn.net/Articles/364546/>): 

[W]e used to have device specific user space interfaces for mouse and keyboard. These caused all sort of problems. A lot of work went into unifying them under evdev. It will be years until the old, messed up interfaces can be totally removed. 

I'm not in favor of repeating the problems with a device specific user space interface for IR. I believe all new input devices should implement the evdev framework. 

In other words, these developers want remote control devices to look like any other input device and generate input events through the same interface. Jon has posted [a proposed IR input driver](<https://lwn.net/Articles/364196/>) for discussion; it is actually a rework of work first posted one year ago. This code moves all processing into the kernel and provides a flexible mechanism for dealing with multiple remote controls. 

As it happens, a number of remote control receivers already work this way, even in the absence of Jon's patch. LIRC is not the sole repository of IR receiver drivers; a fair number of them also live in the mainline kernel already, in the Video4Linux2 subsystem. TV cards often come with a bundled remote control and receiver, so it makes sense to write a driver for the receiver as part of the larger V4L2 driver. These drivers do not use the LIRC interface; instead, they generate input events directly. See [the Conexant CX2388x IR driver](<http://linuxtv.org/hg/v4l-dvb/file/74ad936bcca2/linux/drivers/media/video/cx23885/cx23885-input.c>) for an example of what this sort of driver looks like. 

The discussion covered various approaches to IR receivers without coming to any real resolution. Jon Smirl's [attempt to clarify the goals](<https://lwn.net/Articles/364553/>) for in-kernel IR support may have brought some focus, but little in the way of solid conclusions. Even so, there are some points of near consensus; these include: 

  * There needs to be some sort of API based on the input subsystem, where applications can obtain processed, high-level keycodes for button presses. The goal is to have remote-using applications "just work" whenever possible. 

  * There probably needs to be a separate interface where special-purpose applications can get raw timing data from the receiver - at least, for receivers without built-in decoders which can provide this information. This interface can be used to reverse-engineer the sequences sent by new remote control units and to deal with pathologically-bad hardware. There is talk of funneling raw data through the input layer as well, but it's not clear that doing so buys anything; it may be that just adopting the existing LIRC interface for raw data is as good an approach as any. 

With regard to the keycode interface, there is still a lot of disagreement over where the keycodes should come from. Some developers want all of the IR drivers to be in the kernel, while others are happy with using the LIRC daemon (or something like it) to generate keycodes and push them back into the kernel from user space. In-kernel drivers have the potential to work with no daemon process and they can use the current module loading mechanism. Kernel-based drivers will also have lower response latency than a user-space daemon, saving precious milliseconds for desperate users who want to change channels and evade that "too much information" pharmaceuticals commercial. 

On the other hand, in-kernel drivers are kernel code, with the higher level of risk that always implies. Filtering of input sequences and protocol processing can be a significant amount of work that some would rather see done in user space. It may never be possible to support the more problematic hardware in the kernel. Then, there are the truly wild ideas, such as wiring an IR receiver to a sound card's microphone input - something people actually do, evidently. The fact that some IR protocols [may be patent-encumbered](<https://lwn.net/Articles/364557/>) also needs to be kept in mind. 

Another detail worth bearing in mind: a number of IR receivers are also capable of transmitting information. A solution based solely on the input layer will not be able to handle the output case. 

There is one final, simple point: the LIRC drivers have seen many years of development, and they work. If LIRC is merged directly, the kernel will benefit from that work and the associated lessons learned. If LIRC is dropped in favor of fully in-kernel drivers, chances are good that some of those lessons will have to be learned anew. If the kernel were to go with a non-LIRC approach to IR drivers, it would probably, eventually, reach a point where it had a more capable and flexible system with wider device support than is available now. But, between here and there would be a period - perhaps a long period - where in-kernel IR support was not as good as LIRC. 

Still, that might just be how things go in the end. The kernel development community, always concerned about what it will have to maintain five or ten years in the future, tends not to be in a hurry to merge something now just because it is seen to work. So, while it is yet possible that LIRC could be merged in something close to its current form, it's also possible that it could lurk on the sidelines while something significantly different is created for the mainline.
