---
title: "Video4Linux2 part 2: registration and open()"
url: https://lwn.net/Articles/204545/
date: "October 18, 2006"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: 
---

The [LWN.net Video4Linux2 API series](<http://lwn.net/Articles/203924/>).   
---  
This is the second article in the LWN series on writing drivers for the Video4Linux2 kernel interface; those who have not yet seen [the introductory article](<http://lwn.net/Articles/203924/>) may wish to start there. This installment will look at the overall structure of a Video4Linux driver and the device registration process. 

Before starting, it is worth noting that there are two resources which will prove invaluable for anybody working with video drivers: 

  * The [V4L2 API Specification](<http://v4l2spec.bytesex.org/spec/book1.htm>). This document covers the API from the user-space point of view, but, to a great extent, V4L2 drivers implement that API directly. So most of the structures are the same, and the semantics of the V4L2 calls are clearly laid out. Print a copy (consider cutting out the Free Documentation License text to save trees) and keep it somewhere within easy reach. 

  * The "vivi" driver found in the kernel source as `drivers/media/video/vivi.c`. It is a virtual driver, in that it generates test patterns and does not actually interface to any hardware. As such, it serves as a relatively clear illustration of how V4L2 drivers should be written. 

To start, every V4L2 driver must include the requisite header file: 

```
#include <linux/videodev2.h>
```

Much of the needed information is there. When digging through the headers as a driver author, however, you'll also want to have a look at `include/media/v4l2-dev.h`, which defines many of the structures you'll be working with. 

A video driver will probably have sections which deal with the PCI or USB bus (for example); we'll not spend much time on that part of the driver here. There is often an internal i2c interface, which _will_ be examined later on in this article series. Then, there is the interface to the V4L2 subsystem. That interface is built around `struct video_device`, which represents a V4L2 device. Covering everything that goes into this structure will be the topic of several articles; here we'll just have an overview. 

The `name` field of `struct video_device` is a name for the type of device; it will appear in kernel log messages and in sysfs. The name usually matches the name of the driver. 

There are two fields to describe what type of device is being represented. The first (`type`) looks like a holdover from the Video4Linux1 API; it can have one of four values: 

  * `VFL_TYPE_GRABBER` indicates a frame grabber device - including cameras, tuners, and such. 
  * `VFL_TYPE_VBI` is for devices which pull information transmitted during the video blanking interval. 
  * `VFL_TYPE_RADIO` for radio devices. 
  * `VFL_TYPE_VTX` for videotext devices. 

If your device can perform more than one of the above functions, a separate V4L2 device should be registered for each of the supported functions. In V4L2, however, any of the registered devices can be called upon to function in any of the supported modes. What it comes down to is that, for V4L2, there is really only need for a single device, but compatibility with the older Video4Linux API requires that individual devices be registered for each function. 

The second field, called `type2`, is a bitmask describing the device's capabilities in more detail. It can contain any of the following values: 

  * `VID_TYPE_CAPTURE`: the device can capture video data. 
  * `VID_TYPE_TUNER`: it can tune to different frequencies. 
  * `VID_TYPE_TELETEXT`: it can grab teletext data. 
  * `VID_TYPE_OVERLAY`: it can overlay video data directly into the frame buffer. 
  * `VID_TYPE_CHROMAKEY`: a special form of overlay capability where the video data is only displayed where the underlying frame buffer contains pixels of a specific color. 
  * `VID_TYPE_CLIPPING`: it can clip overlay data. 
  * `VID_TYPE_FRAMERAM`: it uses memory located in the frame buffer device. 
  * `VID_TYPE_SCALES`: it can scale video data. 
  * `VID_TYPE_MONOCHROME`: it is a monochrome-only device. 
  * `VID_TYPE_SUBCAPTURE`: it can capture sub-areas of the image. 
  * `VID_TYPE_MPEG_DECODER`: it can decode MPEG streams. 
  * `VID_TYPE_MPEG_ENCODER`: it can encode MPEG streams. 
  * `VID_TYPE_MJPEG_DECODER`: it can decode MJPEG streams. 
  * `VID_TYPE_MJPEG_ENCODER`: it can encode MJPEG streams. 

Another field initialized by all V4L2 drivers is `minor`, which is the desired minor number for the device. Usually this field will be set to -1, which causes the Video4Linux subsystem to allocate a minor number at registration time. 

There are also three distinct sets of function pointers found within `struct video_device`. The first, consisting of a single function, is the `release()` method. If a device lacks a `release()` function, the kernel will complain (your editor was amused to note that it refers offending programmers to an LWN article). The `release()` function is important: for various reasons, references to a `video_device` structure can remain long after that last video application has closed its file descriptor. Those references can remain after the device has been unregistered. For this reason, it is not safe to free the structure until the `release()` method has been called. So, often, this function consists of a simple `kfree()` call. 

The `video_device` structure contains within it a `file_operations` structure with the usual function pointers. Video drivers will always need `open()` and `release()` operations; note that _this_ `release()` is called whenever the device is closed, not when it can be freed as with the other function with the same name described above. There will often be a `read()` or `write()` method, depending on whether the device performs input or output; note, however, that for streaming video devices, there are other ways of transferring data. Most devices which handle streaming video data will need to implement `poll()` and `mmap()`. And _every_ V4l2 device needs an `ioctl()` method - but they can use `video_ioctl2()`, which is provided by the V4L2 subsystem. 

The third set of methods, stored in the `video_device` structure itself, makes up the core of the V4L2 API. There are several dozen of them, handling various device configuration operations, streaming I/O, and more. 

Finally, a useful field to know from the beginning is `debug`. Setting it to either (or both - it's a bitmask) of `V4L2_DEBUG_IOCTL` and `V4L2_DEBUG_IOCTL_ARG` will yield a fair amount of debugging output which can help a befuddled programmer figure out why a driver and an application are failing to understand each other. 

### Video device registration

Once the `video_device` structure has been set up, it should be registered with: 

```
int video_register_device(struct video_device *vfd, int type, int nr);
```

Here, `vfd` is the device structure, `type` is the same value found in its `type` field, and `nr` is, again, the desired minor number (or -1 for dynamic allocation). The return value should be zero; a negative error code indicates that something went badly wrong. As always, one should be aware that the device's methods can be called immediately once the device is registered; do not call `video_register_device()` until everything is ready to go. 

A device can be unregistered with: 

```
void video_unregister_device(struct video_device *vfd);
```

Stay tuned for the next article in this series, which will begin to look at the implementation of some of these methods. 

### open() and release()

Every V4L2 device will need an `open()` method, which will have the usual prototype: 

```
int (*open)(struct inode *inode, struct file *filp);
```

The first thing an `open()` method will normally do is to locate an internal device corresponding to the given `inode`; this is done by keying on the minor number stored in `inode`. A certain amount of initialization can be performed; this can also be a good time to power up the hardware if it has a power-down option. 

The V4L2 specification defines some conventions which are relevant here. One is that, by design, all V4L2 devices can have multiple open file descriptors at any given time. The purpose here is to allow one application to display (or generate) video data while another one, perhaps, tweaks control values. So, while certain V4L2 operations (actually reading and writing video data, in particular) can be made exclusive to a single file descriptor, the device as a whole should support multiple open descriptors. 

Another convention worth mentioning is that the `open()` method should not, in general, make changes to the operating parameters currently set in the hardware. It should be possible to run a command-line program which configures a camera according to a certain set of desires (resolution, video format, etc.), then run an entirely separate application to, for example, capture a frame from the camera. This mode would not work if the camera's settings were reset in the middle, so a V4L2 driver should endeavor to keep existing settings until an application explicitly resets them. 

The `release()` method performs any needed cleanup. Since video devices can have multiple open file descriptors, `release()` will need to decrement a counter and check before doing anything radical. If the just-closed file descriptor was being used to transfer data, it may necessary to shut down the DMA engine and perform other cleanups. 

The next installment in this series will start into the long process of querying device capabilities and configuring operating modes. Stay tuned.
