---
title: "Free user space for non-graphics drivers"
url: https://lwn.net/Articles/821817/
date: "June 3, 2020"
category: "Development model-User-space ABI"
author: "By Jake Edge June 3, 2020"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
June 3, 2020

In the kernel graphics world, there has been a longstanding "[line in the sand](<https://lwn.net/Articles/394702/>)" that disallows merging kernel drivers without a corresponding free-software user-space driver. The idea is that not having a way to test the full functionality means that the kernel developers cannot verify the proper functioning and security of the driver; changes to the kernel driver may lead to unforeseen (and untestable) problems on the user-space side. More recently, though, we have seen other types of devices with complex drivers, but no useful free user-space piece, that have been proposed for inclusion into the kernel; at least one was merged, but the tide has perhaps turned against those types of drivers at this point—or some of them, anyway. 

In mid-May, Jeffrey Hugo [posted](<https://lwn.net/ml/linux-kernel/1589465266-20056-1-git-send-email-jhugo@codeaurora.org/>) an RFC patch for the "Qualcomm Cloud AI 100" device, which is a PCIe card with an application-specific integrated circuit (ASIC) that targets "deep learning" workloads. The device is also referred to as a QAIC device; it presents a [modem host interface](<https://www.kernel.org/doc/html/latest/mhi/mhi.html>) (MHI) control path and a DMA engine for the data path. These are exposed in the driver as a Linux character device with `ioctl()` commands to access the data path. 

Dave Airlie, who drew the line in the sand back in 2010, [wondered](<https://lwn.net/ml/linux-kernel/CAPM=9txXskVu_yD3DNuR0HgSUsE2v1Pv98dm=AHGvv_z2XKTAQ@mail.gmail.com/>) how the QAIC driver was really any different. It has a user-facing API (which he calls the "uapi") with no open-source users and no tests for it. 

Although this isn't a graphics driver, and Greg [Kroah-Hartman] will likely merge anything to the kernel you throw at him, I do wonder how to validate the uapi from a security perspective. It's always interesting when someone wraps a DMA engine with user ioctls, and without enough information to decide if the DMA engine is secure against userspace misprogramming it. 

Hugo [said](<https://lwn.net/ml/linux-kernel/93238096-5861-c140-b94f-6137977c3d65@codeaurora.org/>) that he would like to put one of the devices ""into the hands of Linaro, so that it can be put into KernelCI"". In order to do that, he needs to get permission from others at Qualcomm. Beyond that, though, it will be difficult to get anything particularly useful on the user-space side: 

Regarding what the community could do on its own, everything but the Linux driver is considered proprietary - that includes the on device firmware and the entire userspace stack. This is a decision above my pay grade. 

I've asked for authorization to develop and publish a simple userspace application that might enable the community to do such testing, but obtaining that authorization has been slow. 

Perhaps to Airlie's surprise, though, Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/20200519174120.GC1158284@kroah.com/>) that he would not review the driver any further without that kind of information (and code); furthermore, he would not merge it and he would actively oppose anyone else merging it, as well. 

Ok, that's a decision you are going to have to push upward on, as we really can't take this without a working, open, userspace. 

Especially given the copyright owner of this code, that would be just crazy and foolish to not have open userspace code as well. Firmware would also be wonderful as well, go poke your lawyers about derivative work issues and the like for fun conversations :) 

Hugo [said](<https://lwn.net/ml/linux-kernel/ce0e69ef-116c-df95-c136-d4714e02e96e@codeaurora.org/>) that he would try, but did not seem to have a lot of hope that he would be able to deliver what the kernel developers are looking for. On the other hand, Daniel Vetter [was surprised](<https://lwn.net/ml/linux-kernel/CAKMK7uG-oP-tcOcNz-ZzTmGondEo-17BCN1kpFBPwb7F8QcM5w@mail.gmail.com/>) by Kroah-Hartman's position: 

So the merge criteria for drivers/accel (atm still drivers/misc but I thought that was interim until more drivers showed up) isn't actually "totally-not-a-gpu accel driver without open source userspace". 

Instead it's "totally-not-a-gpu accel driver without open source userspace" _and_ you have to be best buddies with Greg. Or at least not be on the naughty company list. Since for habanalabs all you wanted is a few test cases to exercise the ioctls. Not the entire userspace. 

Vetter is referring to the [Habana Labs kernel driver](<https://lwn.net/ml/linux-kernel/20190123000057.31477-1-oded.gabbay@gmail.com/>) that was merged over his, Airlie's, and others' objections. The problems that were pointed out at the time are much the same as those raised for the QAIC driver, but it would seem the outcome is going to be different this time. Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/20200520045900.GA2105777@kroah.com/>) that he was particularly concerned with ""the copyright owner of this code"", which must be referring to Qualcomm. The code is actually copyrighted by the Linux Foundation (Kroah-Hartman's employer) and comes from the [Code Aurora Forum](<https://www.codeaurora.org/>), which is a Linux Foundation collaborative project; the Qualcomm Innovation Center (QuIC) is a principal member of that organization. 

He noted that Habana Labs has released its library code as open source since the merge, so that situation has resolved. But he did [admit](<https://lwn.net/ml/linux-kernel/20200520051536.GA2141566@kroah.com/>) that he was wrong about these types of drivers: 

Also, to be fair, I have changed my mind after seeing the mess of complexity that these "ioctls for everyone!" type of pass-through these kinds of drivers are creating. You were right, we need open userspace code in order to be able to properly evaluate and figure out what they are doing is right or not and be able to maintain things over time correctly. 

So I was wrong, and you were right, my apologies for my previous stubbornness. 

The recent submission of a [DirectX driver](<https://en.wikipedia.org/wiki/DirectX>) from Microsoft is in a similar position. Sasha Levin [posted](<https://lwn.net/ml/linux-kernel/20200519163234.226513-1-sashal@kernel.org/>) an RFC patch on May 19: 

The driver exposes a paravirtualized GPU to user mode applications running in a virtual machine on a Windows host. This enables hardware acceleration in environment such as WSL (Windows Subsystem for Linux) where the Linux virtual machine is able to share the GPU with the Windows host. 

The use case, at least for now, is not about graphics at all; it is meant to provide a means for a Linux guest on a Windows system to access the GPU for compute purposes. But the [blog post](<https://devblogs.microsoft.com/directx/directx-heart-linux/>) announcing the feature seems to indicate that graphics support is at least on the horizon. Vetter [said](<https://lwn.net/ml/linux-kernel/CAKMK7uGnSDHdZha-=dZN5ns0sJ2CEnK2693uix4tzqyZb9MXCQ@mail.gmail.com/>) that, as [documented](<https://dri.freedesktop.org/docs/drm/gpu/drm-uapi.html#open-source-userspace-requirements>), a graphics driver needs a free user-space before it can go upstream, but that does not seem to be what is happening here: 

From the blog it sounds like the userspace is all closed. That includes the hw specific part and compiler chunks, all stuff we've generally expected to be able to look [at] in the past for any kind of other driver. 

He suggested that since the driver is not targeting graphics it could move elsewhere in the tree (such as `drivers/hyperv`). Airlie [thought](<https://lwn.net/ml/linux-kernel/CAPM=9txZpiCGkv3jiBC1F8pTe4A2pqWpQDyjRBXk2roFqw+0+Q@mail.gmail.com/>) similarly: 

This is a driver that connects a binary blob interface in the Windows kernel drivers to a binary blob that you run inside a Linux guest. It's a binary transport between two binary pieces. Personally this holds little of interest to me, I can see why it might be nice to have this upstream, but I don't [foresee] any other Linux distributor ever enabling it or having to ship it, it's purely a WSL2 pipe. I'm not saying I'd be happy to see this in the tree, since I don't see the value of maintaining it upstream, but it probably should just [exist] in a drivers/hyperv type area. 

He is concerned, though, that merging it in a Hyper-V-specific part of the tree may lead to trouble down the road if graphics functionality gets added to the driver. But Microsoft's Steve Pronovost [said](<https://lwn.net/ml/linux-kernel/MWHPR21MB0287145CD511A2FF6DA502A9C7B60@MWHPR21MB0287.namprd21.prod.outlook.com/>) that, while graphics support may be in the future, the driver being proposed would not be used; it is strictly meant for the GPU-compute case. There is work going on to support Linux GUI applications on WSL, but that would be done using the [Remote Desktop Protocol](<https://en.wikipedia.org/wiki/Remote_Desktop_Protocol>) (RDP) and a Wayland compositor; that work would be fully open source, he said. 

Levin [noted](<https://lwn.net/ml/linux-kernel/20200519203608.GG33628@sasha-vm/>) that he would personally prefer that the user-space pieces needed for the proposed driver were available as open source ""and wish I could do something about it"". He is amenable to moving the driver outside of the graphics tree, which would seem to be the path forward. He also pointed out that the RFC ""is not a case of 'we want it upstream NOW' but rather 'let's work together to figure out how to do it right' :)"". 

It would seem, then, that the QAIC driver is going to languish, while the Microsoft driver may be able to find its way into the mainline, even though neither has any significant free user-space code that can be used to test it. In part, that may be due to the difficult relationship Qualcomm has had with the kernel development community over the years. Beyond that, the QAIC device seems to be a more complicated beast, with more ways that things can go wrong—invisibly. But the Microsoft driver has not been merged yet; we will have to wait to see whether its lack of a free user space holds it back—or not. 

These are undoubtedly not the last drivers of this sort we will see. There is a trend toward devices that are programmable from user space and the kernel is simply used as a conduit to carry proprietary code and data between user space and the device; only the device maker has any real view into what is actually happening inside it. There is clearly an advantage for device makers to get their drivers into the mainline, but is there any real gain for the community by doing so? As Airlie said about the DirectX driver: ""[...] I don't see the value this adds to the Linux ecosystem at all, and I think it's important when putting a burden on upstream that you provide some value."" 

Users who have these devices may benefit from finding the driver in their distribution kernel, and the device makers get "free maintenance" by the kernel developers, but the community is left with a pile of code that is not particularly useful—and could be fragile in ways that could cause problems in the future. It does not really sound like the usual free-software bargain. Luckily, the kernel developers provide a highly functioning platform that the device makers can use to sell and run their devices; they may just have to maintain and distribute their own drivers in order to do so.
