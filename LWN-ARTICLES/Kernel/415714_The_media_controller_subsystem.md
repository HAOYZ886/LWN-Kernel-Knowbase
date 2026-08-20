---
title: The media controller subsystem
url: https://lwn.net/Articles/415714/
date: "November 16, 2010"
category: "Development model-User-space ABI; Media controller; Video4Linux2"
author: "By Jonathan Corbet November 16, 2010"
---

By **Jonathan Corbet**  
November 16, 2010

Over the course of the last decade, video acquisition hardware has evolved from relatively rare, bulky, external devices to being a standard feature in a large variety of gadgets. Increasingly, chipsets intended for embedded use have video support as a standard feature. This support is becoming more complex; contemporary video devices are not just frame grabbers anymore. That complexity is revealing limitations in the kernel's device model, prompting the proposal of a new "media controller" abstraction. This article will provide an overview and mild critical review of this new subsystem. 

Video acquisition devices have never been entirely simple. Even a minimal camera device will usually be a composite of at least three distinct devices: a sensor, a DMA bridge to move frames between the sensor and main memory, and an I2C bus dedicated to controlling the sensor. Most devices coming onto the market now are more sophisticated than that. For example, the integrated controller in current VIA chipsets (still a very simple device) adds a "high-quality video" (HQV) unit which can perform image rotation and format conversions; that unit can be configured into or out of the processing pipeline depending on the application's needs. For a more complex example, consider the OMAP 3430, which is found in N900 phones; it has multiple video inputs, a white balance processor, a lens shading compensation processor, a resizer, and more. 

Each of these components can be thought of as a separate device which can be powered up or down independently, and which, in some cases, can be configured in or out at any given time. The current V4L2 system wasn't designed with this kind of device structure in mind, and neither was the current Linux device model. An added problem is that these devices can be tied with devices managed by other subsystems - audio devices in particular - making it hard for applications to grasp the whole picture. The media controller is an attempt to rectify that situation. 

The most recent version of the media controller patch was [posted](<https://lwn.net/Articles/408392/>) by Laurent Pinchart back in September; if all goes according to plan, it will be merged for 2.6.38. The patch creates a new `media_device` type which has the responsibility of managing the various components which make up a media-related device. These components are called "entities"; and they can take many forms. Sensors, DMA engines, video processing units, focus controllers, audio devices, and more are all considered to be "entities" in this scheme. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Most entities will have at least one "pad," being a logical connection point where data can flow into or out of the device. "Data" in this sense can be multimedia data, but it might also be a control stream. Pads are exclusively input ("sink") or output ("source") ports, and an entity can have an arbitrary number of each. The final piece is called a "link"; it is a directional connection from a source pad to a sink. Links are created by the media device driver, but they can, in some cases, be enabled or disabled from user space. 

Using this scheme, the simple VIA device described above could be represented with three entities and three links: 

> ![\[Media controller\]](https://static.lwn.net/images/ns/kernel/media-controller.png)

The "sensor" entity has a single source pad which can be connected, via links, to the HQV unit or directly to the DMA controller. Only one of those paths can be active at once. The HQV unit has two pads - one sink, one source - allowing it to be slotted into the video pipeline if need be. The DMA controller has a single sink pad. 

As an aside: entities also have a "group" number assigned to them; groups are intended to indicate hardware which is meant to function together. All of the units described above would probably be placed into the same group by the driver. If there were a microphone attached to the camera, then the associated audio entity would also be placed in the same group. This mechanism is intended to make it easier for applications to associate related devices with each other. 

On the application side, there is a device (probably `/dev/media0` or some such) which can be opened to gain access to this device. From there, the interface looks very much like the rest of V4L2 - lots of `ioctl()` calls to discover what is available and configure it. These calls include: 

  * `MEDIA_IOC_DEVICE_INFO` to get overall information about the device: driver name, device model, etc. 

  * `MEDIA_IOC_ENUM_ENTITIES` is used to iterate through all of the entities contained within the device. Information returned includes an ID number, a coarse entity type (e.g. V4L or ALSA), a subtype (few of these are defined in the patch; "sensor" is one of them), the group ID, the device number, and the numbers of pads and links. 

  * `MEDIA_IOC_ENUM_LINKS` iterates through all of the links attached to source pads on a given entity. Thus, it is only possible to discover the outbound links from any entity; obtaining the whole graph requires iterating through all entities. 

  * `MEDIA_IOC_SETUP_LINK` changes the properties of a specific link; in particular, it can enable or disable the link (though links can be marked "immutable" by the driver). Enabling a link will have the side effect of powering up all components reachable via that link, while disabling the last link to an entity will cause that entity to be powered down. Thus, changing the status of a link affects both the data path and the power configuration of a device. 

Thus far, there have been no applications posted which actually use this framework (though a gstreamer source element is in the works). One can certainly see the utility of being able to discover and modify the configuration of a complex media device in this manner. But, at the Linux Plumbers Conference, your editor heard some concerns that the complexity of this interface could prove daunting to application developers. An application which is intended to work with a specific device (the camera application on a mobile handset, say) can be written with a relatively high level of awareness of that device and make good use of this interface. Writing an application which can make full use of any device - without requiring the developer to know about specific hardware - could be more challenging. 

One other concern raised at LPC was that this functionality should really be exported via sysfs rather than through an `ioctl()`-based API. The information contained here would fit well within a sysfs hierarchy, with links represented by symbolic links in the filesystem. Given that the configuration interface (in its current form) changes a single bit at a time, there is no need for the sort of transactional functionality that can make `ioctl()` preferable to sysfs. On the other hand, V4L2 applications are already a big mass of `ioctl()` calls; the media controller API will be a natural fit while rooting through sysfs would be a new experience for V4L2 developers. 

Something else is worth thinking about here: the problem may be bigger than just media devices. More complex devices are the norm, and it is becoming clear that the kernel's hierarchical device model is not up to the task of representing the structure of our systems. Back in 2009, Rafael Wysocki [proposed a mechanism](<https://lwn.net/Articles/347573/>) for representing power-management dependencies with explicit links. The media controller mechanism looks quite similar; it is even being used for power management purposes. That suggests that we should be looking for a data structure which can represent device connections and dependencies across the kernel, not just in one subsystem. Otherwise we run the risk of creating duplicated structures and multiple user-space ABIs, all of which must be supported indefinitely. 

The media controller subsystem is aimed at solving a real problem, and it is certainly a credible solution. It is also a significant new user-space ABI, one which does not necessarily conform to current ideas of how interfaces should be done. The work done here may also be applicable well beyond the V4L2 and ALSA subsystems, but any attempt at a bigger-picture solution should probably be made before the code is merged and the ABI is set in stone. All of this suggests that the media controller code could benefit from review outside of the V4L mailing list, which tends to be inhabited by relatively focused developers. 

(Thanks to Andy Walls, Hans Verkuil, and Laurent Pinchart for their comments on this article).
