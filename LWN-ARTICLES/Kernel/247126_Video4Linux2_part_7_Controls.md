---
title: "Video4Linux2 part 7: Controls"
url: https://lwn.net/Articles/247126/
date: "August 31, 2007"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: "By Jonathan Corbet August 31, 2007 The LWN.net Video4Linux2 API series ."
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 31, 2007

* * *

The [LWN.net Video4Linux2 API series](<http://lwn.net/Articles/203924/>). 

With the completion of [part 6 of this series](<http://lwn.net/Articles/240667/>), we now know how to set up a video device and transfer frames back and forth. It is a well known fact, however, that users can be hard to please; not content with being able to see video from their camera device, they immediately start asking if they can play with parameters like brightness, contrast, and more. These adjustments could be done in the video application, and sometimes they are, but there are advantages to doing them in the hardware itself when the hardware has that capability. A brightness adjustment, for example, might lose dynamic range if done after the fact, but a hardware-based adjustment may retain the full range that the sensor is capable of delivering. Hardware-based adjustments, obviously, will also be easier on the host processor. 

Current hardware typically has a wide range of parameters which can be adjusted on the fly. Just how those parameters work varies widely from one device to the next, though. An adjustment as simple as "brightness" could involve a straightforward register setting, or it could require a rather more complex change to an obscure transformation matrix. It would be nice to hide as much of this detail from the application as possible, but there are limits to how much hiding can be done. An overly abstract interface might make it impossible to use the hardware's controls to their fullest potential. 

The V4L2 control interface tries to simplify things as much as possible while allowing full use of the hardware. It starts by defining a set of standard control names; these include `V4L2_CID_BRIGHTNESS`, `V4L2_CID_CONTRAST`, `V4L2_CID_SATURATION`, and many more. There are boolean controls for features like white balance, horizontal and vertical mirroring, etc. See [the V4L2 API spec](<http://v4l2spec.bytesex.org/spec/x542.htm>) for a full list of predefined control ID values. There is also a provision for driver-specific controls, but those, clearly, will generally only be usable by special-purpose applications. Private controls start at `V4L2_CID_PRIVATE_BASE` and go up from there. 

In typical fashion, the V4L2 API provides a mechanism by which an application can enumerate the available controls. To that end, they will make `ioctl()` calls which end up in a V4L2 driver via the `vidioc_queryctrl()` callback: 

```
int (*vidioc_queryctrl)(struct file *file, void *private_data,
    			    struct v4l2_queryctrl *qc);
```

The driver will normally fill in the structure `qc` with information about the control of interest, or return `EINVAL` if that control is not supported. This structure has a number of fields: 

```
struct v4l2_queryctrl
        {
    	__u32		     id;
    	enum v4l2_ctrl_type  type;
    	__u8		     name[32];
    	__s32		     minimum;
    	__s32		     maximum;
    	__s32		     step;
    	__s32		     default_value;
    	__u32                flags;
    	__u32		     reserved[2];
        };
```

The control being queried will be passed in via `id`. As a special case, the application can supply a control ID with the `V4L2_CTRL_FLAG_NEXT_CTRL` bit set; when this happens, the driver should return information about the next supported control ID higher than the one given by the application. In any case, `id` should be set to the ID of the control actually being described. 

All of the other fields are set by the driver to describe the selected control. The data type of the control is given in `type`; it can be `V4L2_CTRL_TYPE_INTEGER`, `V4L2_CTRL_TYPE_BOOLEAN`, `V4L2_CTRL_TYPE_MENU` (for a set of fixed choices), or `V4L2_CTRL_TYPE_BUTTON` (for a control which performs some action when set and which ignores any given value). `name` describes the control; it could be used in the interface presented to the user by the application. For integer controls (only), `minimum` and `maximum` describe the range of values implemented by the control, and `step` gives the granularity of that range. `default_value` is exactly what it sounds like - though it is only applicable to integer, boolean, and menu controls. Drivers should set control values to their default at initialization time only; like other device parameters, they should persist across `open()` and `close()` calls. As a result, `default_value` may well not be the current value of the control. 

Inevitably, there is a set of flags which further describe a control. `V4L2_CTRL_FLAG_DISABLED` means that the control is disabled; the application should ignore it. `V4L2_CTRL_FLAG_GRABBED` means that the control, temporarily, cannot be changed, perhaps because another application has taken it over. `V4L2_CTRL_FLAG_READ_ONLY` marks controls which can be queried, but which cannot be changed. `V4L2_CTRL_FLAG_UPDATE` means that adjusting this control may affect the values of other controls. `V4L2_CTRL_FLAG_INACTIVE` marks a control which is not relevant to the current device configuration. And `V4L2_CTRL_FLAG_SLIDER` is a hint that applications should represent the control with a slider-like interface. 

Applications might just query a few controls which have been specifically programmed in, or they may want to enumerate the entire set. In the latter case, they will start at `V4L2_CID_BASE` and step through `V4L2_CID_LASTP1`, perhaps using the `V4L2_CTRL_FLAG_NEXT_CTRL` flag in the process. For controls of the menu variety (type `V4L2_CTRL_TYPE_MENU`), applications will probably want to enumerate the possible values as well. The relevant callback is: 

```
int (*vidioc_querymenu)(struct file *file, void *private_data,
    			    struct v4l2_querymenu *qm);
```

The `v4l2_querymenu` structure looks like: 

```
struct v4l2_querymenu
        {
    	__u32		id;
    	__u32		index;
    	__u8		name[32];
    	__u32		reserved;
        };
```

On input, `id` is the ID value for the menu control of interest, and `index` is the index value for a specific menu value. Index values start at zero and go up to the `maximum` value returned from `vidioc_queryctrl()`. The driver will fill in the `name` of the menu item; the `reserved` field should be set to zero. 

Once the application knows about the available controls, it will likely set about querying and changing their values. The structure used in this case is relatively simple: 

```
struct v4l2_control
        {
    	__u32 id;
    	__s32 value;
        };
```

To query a specific control, an application will set `id` to the ID of the control and make a call which ends up in the driver as: 

```
int (*vidioc_g_ctrl)(struct file *file, void *private_data,
        			 struct v4l2_control *ctrl);
```

The driver should set `value` to the current setting of the control. Of course, it should also be sure that it knows about this specific control and return `EINVAL` if the application attempts to query a nonexistent control. Attempts to query button controls should also return `EINVAL`. 

A request to change a control ends up in: 

```
int (*vidioc_s_ctrl)(struct file *file, void *private_data,
    			 struct v4l2_control *ctrl);
```

The driver should verify the `id` and make sure that `value` falls within the allowed range. If all is well, the new value should be set in the hardware. 

Finally, it is worth noting that there is a separate [extended controls interface](<http://v4l2spec.bytesex.org/spec/x802.htm>) supported with V4L2. This API is meant for relatively complex controls; in practice, its main use is for MPEG encoding and decoding parameters. Extended controls can be grouped into classes, and 64-bit integer values are supported. The interface is similar to the regular control interface; see the API specification for details.
