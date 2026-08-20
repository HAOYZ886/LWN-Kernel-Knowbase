---
title: Coping with complex cameras
url: https://lwn.net/Articles/992411/
date: "October 3, 2024"
category: "Device drivers-Video4Linux2"
author: "By Jonathan Corbet October 3, 2024 LPC"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 3, 2024

* * *

[LPC](<https://lwn.net/Archives/ConferenceIndex/#Linux_Plumbers_Conference-2024>)

Cameras were never the simplest of devices for Linux to support; they have a wide range of operating parameters and can generate high rates of data. In recent years, though, they have become increasingly complex, stressing the ability of the kernel's [media subsystem](<https://docs.kernel.org/userspace-api/media/index.html>) to manage them. At the 2024 [Linux Plumbers Conference](<https://lpc.events/>), developers from that subsystem and beyond gathered to discuss the state of affairs and how complex camera devices should be supported in the future. 

#### The complex-camera summit

Ricardo Ribalda led the session, starting with a summary of the closed-door complex-camera summit that had been held the previous day. "Complex cameras" are the seemingly simple devices that are built into phones, notebooks, and other mobile devices. These devices do indeed collect image data as expected, but as part of that task they also perform an enormous amount of signal processing on the data returned from the sensor. That processing, which can include demosaicing, noise removal, sharpening, white-balance correction, image stabilization, autofocus control, contrast adjustment, high-dynamic-range processing, face recognition, and more, is performed in memory by a configurable pipeline of units collectively known as an image signal processor (ISP). 

[![\[Ricardo Ribalda\]](https://static.lwn.net/images/conf/2024/lpc/RicardoRibalda-sm.png)](<https://lwn.net/Articles/992418/>) Much of this functionality is controlled via a feedback loop that passes through user space. The ISP must be provided with large amounts of data that controls the processing that is to be done; these parameters can add up to 500KB or so of data — for each frame that the processor handles. Naturally, the format of the control data tends to be proprietary and is different for each ISP. 

Ribalda started by saying that the vendors of these devices would like to use the kernel's existing media interface (often called Video4Linux or V4L), but that subsystem was not designed for this type of memory-to-memory ISP. Multiple `ioctl()` calls are required for each frame, adding a lot of overhead. V4L does not provide fences for the management of asynchronous operations; attempts have been made to add fences in the past, but did not succeed. There are no abstractions for advanced scheduling of operations, and no support for modes where multiple buffers are sent to the ISP and it decides which ones are important. Multiple-camera support, needed for modern phones, is lacking in V4L, he said. 

Vendors feel that the V4L API is simply too slow to evolve, so it is failing to keep up with modern camera devices. It is much faster, in the short term at least, for vendors to just bash out a device-specific driver for their hardware. But that leads to multiple APIs for the same functionality, creating fragmentation. 

There has been talk of moving support for these devices over to the direct rendering manager (DRM) subsystem, alongside graphics processors, but the DRM developers are not camera experts. There is also interest in creating pass-through interfaces that let user space communicate directly with the device (a topic that had been [extensively discussed](<https://lwn.net/Articles/990802/>) at the Maintainers Summit a few days before), but allowing such an API requires trusting vendors that have not, in turn, trusted the kernel community enough to provide detailed information about how their devices work. 

The conclusion from the complex-camera summit, Ribalda said, was that the V4L developers would like to see a complete list of technical features needed to support complex camera devices. They would then work to address those shortcomings. If they are unable to do so within a reasonable time, he said, they agreed to not block the incorporation of ISP drivers into the DRM subsystem, possibly with pass-through APIs, instead. 

There are also non-technical challenges, Ribalda continued. It is not just that many aspects of these ISPs are undocumented; vendors claim that they _cannot_ be documented. This area, it seems, is a minefield of patents and "special sauce"; vendors do not want to reveal how their hardware works. But the V4L subsystem currently requires documentation for any feature that users can control from user space. Currently, the policy means that undocumented features cannot be used; changing the policy, though, would likely result in _all_ features becoming undocumented. 

A solution proposed at the summit was to describe a "canonical ISP" with features that all of these devices are expected to have; those features would need to be fully documented and implemented in standard ways. Everything else provided by the ISP could be wired directly to user space via a pass-through interface. That would allow functionality to be made available without requiring documentation of everything the device does. 

Of course, there are problems with this approach, he said. Pass-through interfaces raise security concerns. The kernel does not know what the device is doing and cannot, for example, prevent it from being told to overwrite unrelated data in the kernel. These devices require calibration for every combination of ISP, sensor, and lens; without full documentation, that calibration cannot be done. And, naturally, there will be disagreement over what a canonical ISP should do; vendors will push to implement the bare minimum possible. 

An alternative is to ignore proprietary functionality entirely, and document only the basic functionality of the device. Vendors would then provide an out-of-tree driver to make the device actually work as intended. In this world, though, vendors are unlikely to bother with an upstream driver at all. The result will be hard for both distributors and users to manage. 

#### What to do?

This introduction was followed by an extensive, passionate, wandering, and often loud discussion over the proper approach to take regarding complex cameras. The discussion was also long, far overflowing the allotted time. Rather than trying to capture the whole thing, what follows is an attempt to summarize the most significant points of view. Apologies to all participants whose contribution is not reflected here. 

It was clear that the conclusions from the closed summit did not find a consensus in the room. Almost every aspect of them was questioned at one point or another. Nonetheless, they made a useful starting point for the discussion that followed. 

V4L maintainer Hans Verkuil asserted that, in his experience, the device-specific functionality requested by vendors tends to not really be only found in one device; this functionality should be generalized in the interface, he said. The DRM layer requires [Mesa](<https://www.mesa3d.org/>) support (in user space) for new drivers; that is the level where the API is standardized. V4L, he said, has lacked the equivalent of Mesa, so the kernel API _is_ the standard interface. 

Now, though, the [libcamera](<https://libcamera.org/>) library is taking over the role of providing the "real" interface to camera devices, which can change the situation. Vendor-specific support can be implemented there, hiding it from users. So perhaps the best solution is to require the existence of a libcamera interface for complex camera drivers. Meanwhile, the V4L interface will still be needed to control the sensor part of any processing chain; perhaps code could move to DRM for the ISP part, if V4L support will be too long in coming. Media subsystem maintainer Mauro Carvalho Chehab agreed that libcamera makes a DRM-like model more possible. 

DRM subsystem maintainer Dave Airlie said that the existing architecture of the V4L subsystem is simply not suitable for modern camera devices. It is, he said, in the same position as DRM was 20 years ago, when that subsystem had to make a painful transition from programming device registers to exchanging commands and data with user space via ring buffers. If there is a libcamera API that can describe these devices in a general way, he said, then a kernel driver for a specific device should not be merged until libcamera support is present. Then, he said, the ecosystem will sort itself out over time. 

He later added that he does not want camera drivers in the DRM subsystem, which was not designed for them; he is willing to accept them if need be, though. He had really expected V4L to have evolved some DRM-like capabilities by now; that is where the problem is. 

The lesson from the DRM world, he said, is to just go ahead and build something, and the situation will improve over time. The vendors with better drivers will win in the market. He advised against writing specific rules for the acceptance of drivers, saying that vendors would always try to game them. Instead, each driver should be merged after a negotiation with the vendor, with the requirements being ratcheted up over time. That may mean allowing in some bad code initially, especially from vendors that cooperate early on, but the bar can be raised as the quality of the subsystem improves overall. 

There was some discussion about how closely ISPs actually match the GPUs driven by the DRM subsystem. Sakari Ailus said that they are different; they are a pipeline of processing blocks that is configured by user space. There is only one real command: "process a frame". Libcamera developer Laurent Pinchart said that the current model for ISPs does not involve a ring buffer; instead, user space submits a lot of configuration data for each frame to be processed. Both seemed to think that the DRM approach might not work for this kind of device. 

Daniel Stone said, though, that there are GPUs that operate with similar programming models; they do not all have ring buffers. He took strong exception to the claim that, if pass-through functionality is provided, vendors will have no incentive to upstream their drivers. Often, he said, it's not the vendors who do that work in the first place. The Arm GPU drivers were implemented by Collabora; Arm is only now beginning to help with that work. There were a lot of people who wanted open drivers, he said, and were willing to pay for the work to be done. So he agreed with Airlie that the market would sort things out over time. Manufacturers who participate in the process will do better in the end. 

Several participants disagreed with the premise that vendors need to keep aspects of their hardware secret for patent or competition reasons. As is the case elsewhere in the technology industry, companies are well aware of exactly what their competitors are doing. Airlie added that all of these companies copy each other's work, aided by engineers who move freely between them. 

There was also a fair amount of discussion on whether allowing pass-through drivers would facilitate a complete reverse engineering of the hardware. Pinchart said that the large number of parameters for each frame makes reverse engineering difficult. Stone replied that this argument implies either that ISPs are different from any other device, or that the engineers working with them are less capable than others; neither is true, he said. Airlie added that the same was said about virtual-reality headsets, but then ""one guy figured it out"" and free drivers were created. Pinchart said that regular documentation for these devices would not suffice to write a driver, again to general disagreement. 

#### Goals

The developers in the room tried to at least coalesce on the goals they were trying to reach. The form of the desired API is not clear at this point, though Pinchart said that it should not force a lot of computation in the kernel. Airlie suggested pushing all applications toward the use of libcamera; eventually developers will prefer that over working with proprietary stacks. Pinchart was concerned that allowing pass-through functionality would roll back the gains that have been made with vendors so far. So some features, at least, have to be part of the standard API; the hard part, he said, is deciding which ones. 

Pinchart said there seemed to be a rough agreement that vendors should be required to provide a certain level of functionality to have drivers merged, but he wondered how those requirements should be set. Airlie repeated that hard rules here would be gamed, and that there should be a negotiation with each vendor. If being friendly with one advances the situation, he said, ""go for it"". Pinchart worried that both vendors and the community might object. Airlie said that he just tells complaining vendors to go away, but that the early cooperative vendors, to a great extent, _are_ the community. 

Several developers said that the requirements could vary depending on the market each device serves. For a camera aimed at Chromebooks, open functionality sufficient for basic video conferencing may be sufficient. But for cameras to be used in other settings, the bar may be higher, with requirement for more functionality provided by an in-tree driver. Some developers suggested a minimum image-quality requirement, but that would be hard to enforce; properly measuring image quality requires a well-equipped (and expensive) laboratory. 

As this multi-hour session ran over time, Ribalda made some attempts to distill a set of consensus conclusions, but he was not hugely successful. Nonetheless, this discussion would appear to have made some headway. There are other kernel subsystems that have had to solve this problem in the past, resulting in a lot of experience that can be drawn from. Support for complex camera devices in Linux seems likely to be messy and proprietary for some time but, with luck, it will slowly improve. 

[ Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event. ]
