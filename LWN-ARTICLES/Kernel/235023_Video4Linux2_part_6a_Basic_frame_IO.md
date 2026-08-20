---
title: "Video4Linux2 part 6a: Basic frame I/O"
url: https://lwn.net/Articles/235023/
date: "May 18, 2007"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: 
---

The [LWN.net Video4Linux2 API series](<http://lwn.net/Articles/203924/>).   
---  
This [series of articles](<http://lwn.net/Articles/203924/>) on video drivers has been through several installments, but we have yet to transfer a single frame of video data. At this point, though, we have covered enough of the format negotiation details that we can begin to look at how video frames move between the application and device. 

The Video4Linux2 API defines three different ways of transferring video frames, two of which are actually available in the current implementation: 

  * The `read()` and `write()` system calls can be used in the normal way. Depending on the hardware and how the driver is implemented, this technique might be relatively slow - but it does not have to be that way. 

  * Frames can be streamed directly to and from buffers accessible to the application. Streaming is usually the most efficient way to move video data; this interface also allows for the transfer of some useful metadata with the image frames. There are two variants of the streaming technique, depending on whether the buffers are located in user or kernel space. 

  * The Video4Linux2 API specification provides for an asynchronous I/O mechanism for frame transfer. This mode has not been implemented, however, and cannot be used. 

This article will look at the simple `read()` and `write()` interface; streaming transfers will be covered in the next installment. 

### read() and write()

Implementation of `read()` and `write()` is not required by the Video4Linux2 specification. Many simpler applications expect these system calls to be available, though, so, if possible, the driver writer should make them work. If the driver does support these calls, it should be sure to set the `V4L2_CAP_READWRITE` bit in response to a `VIDIOC_QUERYCAP` call (described in [part 3](<http://lwn.net/Articles/206765/>)). In your editor's experience, however, most applications do not bother to check whether these calls are available before attempting to use them. 

The driver's `read()` and/or `write()` methods must be stored in the `fops` field of the associated `video_device` structure. Note that the Video4Linux2 specification requires drivers implementing these methods to provide a `poll()` operation as well. 

A naive implementation of `read()` on a frame grabber device is straightforward: the driver tells the hardware to start capturing frames, delivers one to the user-space buffer, stops the hardware, and returns. If possible, the driver should arrange for the DMA operation to transfer the data directly to the destination buffer, but that is only possible if the controller can handle scatter/gather I/O. Otherwise, the driver will need to buffer the frame through the kernel. Similarly, write operations should go directly to the device if possible, but be buffered through the kernel otherwise. 

Less simplistic implementations are possible. Your editor's "Cafe" driver, for example, leaves the camera controller running in a speculative mode after a `read()` operation. For the next fraction of a second, subsequent frames from the camera will be buffered in the kernel; if the application issues another `read()` call, it will be satisfied more quickly without the need to start up the hardware again. After a number of unclaimed frames the controller is put back into an idle state. Similarly, a `write()` operation could delay the first frame by a few tens of milliseconds with the idea of helping the application stream frames at the hardware's expected rate. 

### Streaming parameters

The `VIDIOC_G_PARM` and `VIDIOC_S_PARM` `ioctl()` calls adjust some parameters which are specific to `read()` and `write()` implementations - and some which are more general. It appears to be a call where miscellaneous options with no obvious home were put. We'll cover it here, even though some of the parameters affect streaming I/O as well. 

Video4Linux2 drivers supporting these calls provide the following two methods: 

```
int (*vidioc_g_parm) (struct file *file, void *private_data,
        			  struct v4l2_streamparm *parms);
        int (*vidioc_s_parm) (struct file *file, void *private_data,
    			  struct v4l2_streamparm *parms);
```

The `v4l2_streamparm` structure contains one of those unions which should be getting familiar to readers of this series by now: 

```
struct v4l2_streamparm
        {
    	enum v4l2_buf_type type;
    	union
    	{
    		struct v4l2_captureparm	capture;
    		struct v4l2_outputparm	output;
    		__u8 raw_data[200];
    	} parm;
        };
```

The `type` field describes the type of operation to be affected; it will be `V4L2_BUF_TYPE_VIDEO_CAPTURE` for capture devices and `V4L2_BUF_TYPE_VIDEO_OUTPUT` for output devices. It can also be `V4L2_BUF_TYPE_PRIVATE`, in which case the `raw_data` field is used to pass some sort of private, non-portable, probably discouraged data through to the driver. 

For capture devices, the `parm.capture` field will be of interest. That structure looks like this: 

```
struct v4l2_captureparm
        {
    	__u32		   capability;
    	__u32		   capturemode;
    	struct v4l2_fract  timeperframe;
    	__u32		   extendedmode;
    	__u32              readbuffers;
    	__u32		   reserved[4];
        };
```

`capability` is a set of capability flags; the only one currently defined is `V4L2_CAP_TIMEPERFRAME` which indicates that the device can vary its frame rate. `capturemode` is another flag field with exactly one flag defined: `V4L2_MODE_HIGHQUALITY`, intended to put the hardware into a high-quality mode suitable for single-frame captures. This mode can make any number of sacrifices (in terms of the data formats supported, exposure times, etc.) in order to get the best image quality that the device can handle. 

The `timeperframe` field is used to specify the desired frame rate. It is yet another structure: 

```
struct v4l2_fract {
    	__u32   numerator;
    	__u32   denominator;
        };
```

The quotient described by `numerator` and `denominator` gives the time between successive frames on the device. Another driver-specific field is `extendedmode`, which has no defined meaning in the API. The `readbuffers` field is the number of buffers the kernel should use for incoming frames when the `read()` method is being used. 

For video output devices, the structure looks like: 

```
struct v4l2_outputparm
        {
    	__u32		   capability;
    	__u32		   outputmode;
    	struct v4l2_fract  timeperframe;
    	__u32		   extendedmode;
    	__u32              writebuffers;
    	__u32		   reserved[4];
        };
```

The `capability`, `timeperframe`, and `extendedmode` fields are exactly the same as for capture devices. `outputmode` and `writebuffers` have the same effect as `capturemode` and `readbuffers`, respectively. 

When the application wishes to query the current parameters, it will issue a `VIDIOC_G_PARM` call, resulting in a call to the driver's `vidioc_g_parm()` method. The driver should provide the current settings, being sure to set the `extendedmode` field to zero if it is not being used, and the `reserved` field to zero always. 

An attempt to set the parameters results in a call to `vidioc_s_parm()`. In this case, the driver should set the parameters as closely as possible to the application's request and adjust the `v4l2_streamparm` structure to reflect the values which were actually used. For example, the application might request a higher frame rate than the hardware can provide; in this case, the fastest possible rate should be programmed and the `timeperframe` field set to the actual frame rate. 

If `timeperframe` is given as zero by the application, the driver should program the nominal frame rate associated with the current video norm. If `readbuffers` or `writebuffers` is zero, the driver should return the current settings rather than getting rid of the current buffers. 

At this point, we have covered enough to write a simple driver supporting frame transfer with `read()` or `write()`. Most serious applications will want to use streaming I/O, however: the streaming mode makes higher performance easier, and it allows frames to be packaged with relevant metadata like sequence numbers. Tune in for the next installment in this series which will discuss how to implement the streaming API in video drivers.
