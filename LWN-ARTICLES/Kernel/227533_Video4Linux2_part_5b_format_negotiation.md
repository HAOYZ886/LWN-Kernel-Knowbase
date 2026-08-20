---
title: "Video4Linux2 part 5b: format negotiation"
url: https://lwn.net/Articles/227533/
date: "March 23, 2007"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: 
---

The [LWN.net Video4Linux2 API series](<http://lwn.net/Articles/203924/>).   
---  
This article is a continuation of the irregular LWN series on writing video drivers for Linux. The [introductory article](<http://lwn.net/Articles/203924/>) describes the series and contains pointers to the previous articles. In [the last episode](<http://lwn.net/Articles/218798/>), we looked at how the Video4Linux2 API describes video formats: image sizes and the representation of pixels within them. This article will complete the discussion by describing the process of coming to an agreement with an application on an actual video format supported by the hardware. 

As we saw in the previous article, there are many ways of representing image data in memory. There is probably no video device on the market which can handle all of the formats understood by the Video4Linux interface. Drivers are not expected to support formats not understood by the underlying hardware; in fact, performing format conversions within the kernel is explicitly frowned upon. So the driver must make it possible for the application to select a format which works with the hardware. 

The first step is to simply allow the application to query the supported formats. The `VIDIOC_ENUM_FMT` `ioctl()` is provided for the purpose; within the driver this command turns into a call to this callback (if a video capture device is being queried): 

```
int (*vidioc_enum_fmt_cap)(struct file *file, void *private_data,
    			       struct v4l2_fmtdesc *f);
```

This callback will ask a video capture device to describe one of its formats. The application will pass in a `v4l2_fmtdesc` structure: 

```
struct v4l2_fmtdesc
        {
    	__u32		    index;
    	enum v4l2_buf_type  type;
    	__u32               flags;
    	__u8		    description[32];
    	__u32		    pixelformat;
    	__u32		    reserved[4];
        };
```

The application will set the `index` and `type` fields. `index` is a simple integer used to identify a format; like the other indexes used by V4L2, this one starts at zero and increases to the maximum number of formats supported. An application can enumerate all of the supported formats by incrementing the index value until the driver returns `EINVAL`. The `type` field describes the data stream type; it will be `V4L2_BUF_TYPE_VIDEO_CAPTURE` for a video capture (camera or tuner) device. 

If the `index` corresponds to a supported format, the driver should fill in the rest of the structure. The `pixelformat` field should be the fourcc code describing the video representation and `description` a short textual description of the format. The only defined value for the `flags` field is `V4L2_FMT_FLAG_COMPRESSED`, which indicates a compressed video format. 

The above callback is for video capture devices; it will only be called when `type` is `V4L2_BUF_TYPE_VIDEO_CAPTURE`. The `VIDIOC_ENUM_FMT` call will be split out into different callbacks depending on the type field: 

```
/* V4L2_BUF_TYPE_VIDEO_OUTPUT */
        int (*vidioc_enum_fmt_video_output)(file, private_date, f);
    
        /* V4L2_BUF_TYPE_VIDEO_OVERLAY */
        int (*vidioc_enum_fmt_overlay)(file, private_date, f);
    
        /* V4L2_BUF_TYPE_VBI_CAPTURE */
        int (*vidioc_enum_fmt_vbi)(file, private_date, f);
    
        /* V4L2_BUF_TYPE_SLICED_VBI_CAPTURE */ */
        int (*vidioc_enum_fmt_vbi_capture)(file, private_date, f);
    
        /* V4L2_BUF_TYPE_VBI_OUTPUT */
        /* V4L2_BUF_TYPE_SLICED_VBI_OUTPUT */
        int (*vidioc_enum_fmt_vbi_output)(file, private_date, f);
    
        /* V4L2_BUF_TYPE_VIDEO_PRIVATE */
        int (*vidioc_enum_fmt_type_private)(file, private_date, f);
```

The argument types are the same for all of these calls. It's worth noting that drivers can support special buffer types with codes starting with `V4L2_BUF_TYPE_PRIVATE`, but that would clearly require a special understanding on the application side. For the purposes of this article, we will focus on video capture and output devices; the other types of video devices will be examined in future installments. 

The application can find out how the hardware is currently configured with the `VIDIOC_G_FMT` call. The argument passed in this case is a `v4l2_format` structure: 

```
struct v4l2_format
        {
    	enum v4l2_buf_type type;
    	union
    	{
    		struct v4l2_pix_format		pix;
    		struct v4l2_window		win;
    		struct v4l2_vbi_format		vbi;
    		struct v4l2_sliced_vbi_format	sliced;
    		__u8	raw_data[200];
    	} fmt;
        };
```

Once again, `type` describes the buffer type; the V4L2 layer will split this call into one of several driver callbacks depending on that type. For video capture devices, the callback is: 

```
int (*vidioc_g_fmt_cap)(struct file *file, void *private_data,
        			    struct v4l2_format *f);
```

For video capture (and output) devices, the `pix` field of the union is of interest. This is the `v4l2_pix_format` structure seen in the previous installment; the driver should fill in that structure with the current hardware settings and return. This call should not normally fail unless something is seriously wrong with the hardware. 

The other callbacks are: 

```
int (*vidioc_s_fmt_overlay)(file, private_data, f);
        int (*vidioc_s_fmt_video_output)(file, private_data, f);
        int (*vidioc_s_fmt_vbi)(file, private_data, f);
        int (*vidioc_s_fmt_vbi_output)(file, private_data, f);
        int (*vidioc_s_fmt_vbi_capture)(file, private_data, f);
        int (*vidioc_s_fmt_type_private)(file, private_data, f);
```

The `vidioc_s_fmt_video_output()` callback uses the same `pix` field in the same way as capture interfaces do. 

Most applications will eventually want to configure the hardware to provide a format which works for their purpose. There are two interfaces provided for changing video formats. The first of these is the `VIDIOC_TRY_FMT` call, which, within a V4L2 driver, turns into one of these callbacks: 

```
int (*vidioc_try_fmt_cap)(struct file *file, void *private_data,
    			      struct v4l2_format *f);
        int (*vidioc_try_fmt_video_output)(struct file *file, void *private_data,
    			      	       struct v4l2_format *f);
        /* And so on for the other buffer types */
```

To handle this call, the driver should look at the requested video format and decide whether that format can be supported by the hardware or not. If the application has requested something impossible, the driver should return `-EINVAL`. So, for example, a fourcc code describing an unsupported format or a request for interlaced video on a progressive-only device would fail. On the other hand, the driver can adjust size fields to match an image size supported by the hardware; normal practice is to adjust sizes downward if need be. So a driver for a device which only handles VGA-resolution images would change the `width` and `height` parameters accordingly and return success. The `v4l2_format` structure will be copied back to user space after the call; the driver should update the structure to reflect any changed parameters so the application can see what it is really getting. 

The `VIDIOC_TRY_FMT` handlers are optional for drivers, but omitting this functionality is not recommended. If provided, this function is callable at any time, even if the device is currently operating. It should _not_ make any changes to the actual hardware operating parameters; it is just a way for the application to find out what is possible. 

When the application wants to change the hardware's format for real, it does a `VIDIOC_S_FMT` call, which arrives at the driver in this form: 

```
int (*vidioc_s_fmt_cap)(struct file *file, void *private_data,
        			    struct v4l2_format *f);
        int (*vidioc_s_fmt_video_output)(struct file *file, void *private_data,
        			             struct v4l2_format *f);
```

Unlike `VIDIOC_TRY_FMT`, this call cannot be made at arbitrary times. If the hardware is currently operating, or if it has streaming buffers allocated (a topic for yet another future installment), changing the format could lead to no end of mayhem. Consider what happens, for example, if the new format is larger than the buffers which are currently in use. So the driver should always ensure that the hardware is idle and fail the request (with `-EBUSY`) if not. 

A format change should be atomic - it should change all of the parameters to match the request or none of them. Once again, image size parameters can be adjusted by the driver if need be. The usual form of these callbacks is something like this: 

```
int my_s_fmt_cap(struct file *file, void *private, 
                         struct v4l2_format *f)
        {
    	struct mydev *dev = (struct mydev *) private;
    	int ret;
    
    	if (hardware_busy(mydev))
    	    return -EBUSY;
    	ret = my_try_fmt_cap(file, private, f);
    	if (ret != 0)
    	    return ret;
    	return tweak_hardware(mydev, &f->fmt.pix);
        }
```

Using the `VIDIOC_TRY_FMT` handler avoids duplication of code and gets rid of any excuse for not implementing that handler in the first place. If the "try" function succeeds, the resulting format is known to work and can be programmed directly into the hardware. 

There are a number of other calls which influence how video I/O is done. Future articles will look at some of them. Support for setting formats is enough to enable applications to start transferring images, however, and that is what the purpose of all this structure is in the end. So the next article, hopefully to come after a shorter delay than happened this time around, will get into support for reading and writing video data.
