---
title: Requirements for accelerator drivers
url: https://lwn.net/Articles/870418/
date: "September 27, 2021"
category: "Device drivers-Accelerators"
author: "By Jonathan Corbet September 27, 2021 Maintainers summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 27, 2021

* * *

[Maintainers summit](<https://lwn.net/Articles/870415/>)

In August, a long-running dispute over drivers for AI accelerators [flared up](<https://lwn.net/Articles/867168/>) in the kernel community. Drivers for graphics accelerators are required to have at least one open-source implementation of the user-space side of the driver (which is where most of the logic is). Drivers for other types of accelerators have not, so far, been held to that same standard, which has created some friction within the community and an inconsistent experience for developers. The 2021 Maintainers Summit took up this issue in the hope of creating a more coherent policy. 

Greg Kroah-Hartman is the subsystem maintainer who has accepted a number of accelerator drivers without applying the open-user-space standard. He started off the session by saying that he can't tell developers of these drivers "no" when there are no standard requirements he can point them to. Dave Airlie, the DRM (graphics subsystem) maintainer, said that his subsystem does indeed have those kinds of standards, but acknowledged that it is a lot to ask that those standards be applied generally. Saying "no", he said, is the best way to get developers to put in the effort to do things right; if the bar is set too low, developers drop their code in, then disappear. Saying "no" makes them engage more. 

We need, he continued, to be more responsible for the bigger picture, and that means that we need information about how the hardware the kernel drives actually works. That becomes especially true for drivers that use certain parts of the kernel API, and the DMA-BUF API in particular. DMA-BUF is a mechanism for drivers to interface with each other; a new driver using that API will be talking to other complex drivers that "have been through all the hoops". There is no desire to compromise the operation of those drivers through interaction with a new driver whose developers have not joined the community. 

The Habana AI-accelerator driver, which is what has set off most of the controversy, is actually better than most, Airlie said. But it will still create security problems. Developers of drivers like this are not experts on creating secure kernel APIs. Kroah-Hartman said that, if drivers like the Habana one are kept out of the kernel, they'll still use APIs like DMA-BUF, nobody will see it, and the result will be far worse. But Airlie repeated that DMA-BUF is a line he does not want to see crossed. 

Kroah-Hartman said he could understand this rule for graphics drivers, but for drivers like Habana's there is no standard that he can apply. Airlie answered that the Habana accelerator is a GPU at heart; one could implement [OpenCL](<https://developer.nvidia.com/opencl>) on top of it — something he didn't know until Habana open-sourced its compiler. If a vendor is making a compute-only graphics card, he asked, why should they have to jump through hoops when their competitors don't? 

So, Kroah-Hartman asked, where is the line where drivers need to come with an open-source user-space implementation? Airlie said that the Habana driver was put forward as "not a GPU driver", but now it is using DMA-BUF. That is where the line should be. This is the standard that the InfiniBand subsystem is using as well now. 

Arnd Bergmann said that there are a couple of cases here. For accelerators that can run anything, like GPUs can, it makes sense for the drivers to go through the DRM subsystem and adhere to DRM's rules. For devices with a more defined purpose, though, it might be better for them to come in with a custom kernel interface and lower requirements. 

Kroah-Hartman said that maintainers have to make value judgments; a lot of new subsystems are submitted to him and he needs to make a decision on each. What should he do? Airlie reiterated that the line should be drawn at the DMA-BUF and DMA fence APIs. "A little driver sitting in a corner" can be merged without a lot of rules, but accelerators inevitably reach the point where they need to use DMA-BUF. There is no point in running an accelerator without access to DMA or graphics. These devices start simple, he said, but once they go toward production they get more complex. Kroah-Hartman agreed to uphold the DMA-BUF line. 

Airlie said that one problem with graphics is that there is no common user-space API for all GPUs, just "a small discoverability API". These devices are so different that any attempt to create a standard API would just get in the way. But that means there is little control over the API that any given driver provides. As it is, graphics developers are often finding interfaces in the drivers that user space has never made use of. 

The DMA-BUF line is good, he said, because using that API brings developers into contact with the experts. There is great value in having a community developer who knows where the user-space code for a device is; that makes it possible to see which interfaces are actually needed, among other things. Vendors need to release the compiler for their devices so that the DRM developers can see what the hardware is capable of. If the device can perform DMA directly, for example, the API has to prevent users from accessing that capability. 

#### Torvalds speaks

At this point, Linus Torvalds unmuted to say that he wanted to argue against some of the points that Airlie was making. Airlie is coming from a subsystem where the community has 25 years of experience; there is history and a community. The developers know how these devices will work. When new people come, though, we don't want to create a high bar to entry. Yes, they will do things wrong the first time, but until we let them into the kernel, they will not learn how to do things right. 

For this reason, Torvalds is in favor of accepting new code and letting different groups make their own mistakes. The DRM developers, after all, screwed a lot of things up badly; that was part of the path toward their current API. The DRM developers certainly didn't come in knowing all that they know now. The same will be true of companies like Habana; they will do things wrong, but if we block them, they will never get things right. That also is why he let ksmbd and the ntfs3 filesystem in for 5.15. 

In other words, he continued, the community should be open to taking in new subsystems, but should also be more proactive in throwing them out again. If a subsystem causes problems for others, out it goes. If there is no user space for it, why keep it around? If the Habana driver creates trouble, it can be thrown out. 

The security issue, he said, is "pure bullshit". Hardware engineers already own the machine, and we cannot protect ourselves against what a device might do; we should worry more about maintainability than security. We can't fix hardware-level security problems, but we can try to ensure maintainability. Torvalds did say, though, that Airlie was correct when it comes to use of DMA-BUF. Drivers are mostly independent, but use of DMA-BUF is the point where they start to interact with each other, and that can bring in maintainability problems. 

#### Tests

Kees Cook spoke up to say that code quality is the real issue here, and that we just don't have enough automated analysis to assure that quality anywhere in the kernel. Where are the tests for ksmbd, he asked? In general, there is no way to find out where the tests (if any) for a given body of code are. He is not a big fan of BPF, he said, but the BPF verifier is comprehensive and prevents a lot of problems. 

Ted Ts'o said that the syzbot fuzzing system can be annoying, but it is also great. Network filesystems (like ksmbd) are missing a good fuzzing solution. Accelerators, each of which has its own instruction-set architecture, will be an even bigger problem. Airlie, though, said that tests don't help if nothing uses the API that the tests are exercising. They don't help with maintainability, since they can't tell maintainers which APIs are actively in use. That is why the DRM developers insist on an active user space, something with life and developers who can answer questions. 

Chris Mason said that, as somebody who has been pushing vendors to get their code into the kernel, he feels that the community is creating an environment where the NVIDIA model (proprietary drivers) is the most efficient way to go. The harder it is to get code upstream, the harder it is to convince vendors to do the right thing. So lowering the bar seems like the right thing to do. We have to let vendors go though the process of doing things wrong and feel the associated pain. 

Torvalds talked about a recent conversation with a large company, where he was surprised to learn that some core developers employed there don't always want to work upstream. They tend to be worried about the one workload they care about, so they hack on that and don't worry about the larger case. "Upstream first" is a goal, he said, but it cannot be a hard requirement. We want developers to want to work with us; we don't want to be the straightjacket that they have to work within. That means we have to be somewhat flexible. 

Kroah-Hartman said that he wants to take new stuff; that makes its developers a part of the community and gets them to care about us. Airlie replied that vendors only care about customers, so they will only care about us if their customers do. Kroah-Hartman continued that we all have to work together and be accepting; we can't force the creation of a unified API across competitors before the code is merged. Mason rephrased that as "we can't make them drink the Kool-Aid if we don't let them into the restaurant". 

Ts'o brought this extended discussion to a close by asking if the group was converging on any sort of consensus. Is there anything that can be said to unify the criteria for driver acceptance? Airlie said that there seemed to be agreement on drawing the line at use of DMA-BUF and the fence API, and nobody disagreed with him. There was a bit of trailing discussion on how to notice when that line was crossed. I suggested moving those interfaces into a [module namespace](<https://lwn.net/Articles/760045/>), and Kroah-Hartman [posted a patch](<https://lwn.net/ml/linux-kernel/YU8oVDFoeD5YYeDT@kroah.com/>) to the effect a few days later.
