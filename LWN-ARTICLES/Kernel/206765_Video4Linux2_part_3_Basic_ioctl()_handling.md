---
title: "Video4Linux2 part 3: Basic ioctl() handling"
url: https://lwn.net/Articles/206765/
date: "October 30, 2006"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: 
---

The [LWN.net Video4Linux2 API series](<http://lwn.net/Articles/203924/>).   
---  
Anybody who has spent any amount of time working through the [Video4Linux2 API specification](<http://v4l2spec.bytesex.org/spec/book1.htm>) will have certainly noted that V4L2 makes heavy use of the `ioctl()` interface. Perhaps more than just about any other type of peripheral, video hardware has a vast number of knobs to tweak. Video streams have many parameters associated with them, and, often, there is quite a bit of processing done in the hardware. Trying to operate video hardware outside of its well-supported modes can lead to poor performance at best, and often no performance at all. So there is no alternative to exposing many of the hardware's features and quirks to the end application. 

Traditionally, video drivers have included `ioctl()` functions of approximately the same length as a Neal Stephenson novel; while the functions often come to more satisfying conclusions than the novels, they do tend to drag a lot in the middle. So the V4L2 API was changed in 2.6.18; the interminable `ioctl()` function has been replaced with a large set of callbacks which implement the individual `ioctl()` functions. There are, in fact, 79 of them in 2.6.19-rc3. Fortunately, most drivers need not implement all - or even most - of the possible callbacks. 

What has really happened is that the long `ioctl()` function has been moved into `drivers/media/video/videodev.c`. This code handles the movement of data between user and kernel space and dispatches individual `ioctl()` calls to the driver. To use it, the driver need only use `video_ioctl2()` as its `ioctl()` method in the `video_device` structure. Actually, most drivers should be able to use it as `unlocked_ioctl()` instead; the locking within the Video4Linux2 layer can handle it, and drivers should have proper locking in place as well. 

The first callback your driver is likely to implement is: 

```
int (*vidioc_querycap)(struct file *file, void *priv, 
                               struct v4l2_capability *cap);
```

This function handles the `VIDIOC_QUERYCAP` `ioctl()`, which asks a simple "who are you and what can you do?" question. Implementing it is mandatory for V4L2 drivers. In this function, as with all other V4L2 callbacks, the `priv` argument is the contents of `file->private_data` field; the usual practice is to point it at the driver's internal structure representing the device at `open()` time. 

The driver should respond by filling in the structure `cap` and returning the usual "zero or negative error code" value. On successful return, the V4L2 layer will take care of copying the response back into user space. 

The `v4l2_capability` structure (defined in `<linux/videodev2.h>`) looks like this: 

```
struct v4l2_capability
        {
    	__u8	driver[16];	/* i.e. "bttv" */
    	__u8	card[32];	/* i.e. "Hauppauge WinTV" */
    	__u8	bus_info[32];	/* "PCI:" + pci_name(pci_dev) */
    	__u32   version;        /* should use KERNEL_VERSION() */
    	__u32	capabilities;	/* Device capabilities */
    	__u32	reserved[4];
        };
```

The `driver` field should be filled in with the name of the device driver, while the `card` field should have a description of the hardware behind this particular device. Not all drivers bother with the `bus_info` field; those that do usually use something like: 

```
sprintf(cap->bus_info, "PCI:%s", pci_name(&my_dev));
```

The `version` field holds a version number for the driver. The `capabilities` field is a bitmask describing various things that the driver can do: 

  * `V4L2_CAP_VIDEO_CAPTURE`: The device can capture video data. 
  * `V4L2_CAP_VIDEO_OUTPUT`: The device can perform video output. 
  * `V4L2_CAP_VIDEO_OVERLAY`: It can do video overlay onto the frame buffer. 
  * `V4L2_CAP_VBI_CAPTURE`: It can capture raw video blanking interval data. 
  * `V4L2_CAP_VBI_OUTPUT`: It can do raw VBI output. 
  * `V4L2_CAP_SLICED_VBI_CAPTURE`: It can do sliced VBI capture. 
  * `V4L2_CAP_SLICED_VBI_OUTPUT`: It can do sliced VBI output. 
  * `V4L2_CAP_RDS_CAPTURE`: It can capture Radio Data System (RDS) data. 
  * `V4L2_CAP_TUNER`: It has a computer-controllable tuner. 
  * `V4L2_CAP_AUDIO`: It can capture audio data. 
  * `V4L2_CAP_RADIO`: It is a radio device. 
  * `V4L2_CAP_READWRITE`: It supports the `read()` and/or `write()` system calls; very few devices will support both. It makes little sense to write to a camera, normally. 
  * `V4L2_CAP_ASYNCIO`: It supports asynchronous I/O. Unfortunately, the V4L2 layer as a whole does not yet support asynchronous I/O, so this capability is not meaningful. 
  * `V4L2_CAP_STREAMING`: It supports `ioctl()`-controlled streaming I/O. 

The final field (`reserved`) should be left alone. The V4L2 specification requires that `reserved` be set to zero, but, since `video_ioctl2()` sets the entire structure to zero, that is nicely taken care of. 

A fairly typical implementation can be found in the "vivi" driver: 

```
static int vidioc_querycap (struct file *file, void  *priv,
    					struct v4l2_capability *cap)
        {
    	strcpy(cap->driver, "vivi");
    	strcpy(cap->card, "vivi");
    	cap->version = VIVI_VERSION;
    	cap->capabilities =	V4L2_CAP_VIDEO_CAPTURE |
    				V4L2_CAP_STREAMING     |
    				V4L2_CAP_READWRITE;
    	return 0;
        }
```

Given the presence of this call, one would expect that applications would use it and avoid asking specific devices to perform functions that they are not capable of. In your editor's limited experience, however, applications tend not to pay much attention to the `VIDIOC_QUERYCAP` call. 

Another callback, which is optional and not often implemented, is: 

```
int (*vidioc_log_status) (struct file *file, void *priv);
```

This function, implementing `VIDIOC_LOG_STATUS`, is intended to be a debugging aid for video application writers. When called, it should print information describing the current status of the driver and its hardware. This information should be sufficiently verbose to help a confused application developer figure out why the video display is coming up blank. Your editor would also recommend, however, that it be moderated with a call to `printk_ratelimit()` to keep it from being used to slow the system and fill the logfiles with junk. 

The next installment will start in on the remaining 77 callbacks. In particular, we will begin to look at the long process of negotiating a set of operating modes with the hardware.
