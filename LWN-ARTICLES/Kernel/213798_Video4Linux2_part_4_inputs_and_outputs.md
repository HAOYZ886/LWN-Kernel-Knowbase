---
title: "Video4Linux2 part 4: inputs and outputs"
url: https://lwn.net/Articles/213798/
date: "December 13, 2006"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: 
---

The [LWN.net Video4Linux2 API series](<http://lwn.net/Articles/203924/>).   
---  
This is the fourth article in the irregular LWN series on writing video drivers for Linux. Those who have not yet read [the introductory article](<http://lwn.net/Articles/203924/>) may want to start there. This week's episode describes how an application can determine which inputs and outputs are available on a given adapter and select between them. 

In many cases, a video adapter does not provide a lot of input and output options. A camera controller, for example, may provide the camera and little else. In other cases, however, the situation is more complicated. A TV card might have multiple inputs corresponding to different connectors on the board; it could even have multiple tuners capable of functioning independently. Sometimes those inputs have different characteristics; some might be able to tune to a wider range of video standards than others. The same holds for outputs. 

Clearly, for an application to be able to make full use of a video adapter, it must be able to find out about the available inputs and outputs, and it must be able to select the one it wishes to operate with. To that end, the Video4Linux2 API offers three different `ioctl()` calls for dealing with inputs, and an equivalent three for outputs. Drivers should implement all three (for each functionality supported by the hardware), even though, for simple hardware, the corresponding code can be quite simple. Drivers should also provide reasonable defaults on startup. What a driver should not do, however, is reset input and output information when an application exits; as with other video parameters, these settings should be left unchanged between opens. 

### Video standards

Before we can get into the details of inputs and outputs, however, we must have a look at video standards. These standards describe how a video signal is formatted for transmission - resolution, frame rates, etc. These standards are usually set by regulatory authorities in each country. There are three major types of video standard used in the world: NTSC (used in North America, primarily), PAL (much of Europe, Africa, and Asia), and SECAM (France, Russia, parts of Africa). There are, however, variations in the standards from one country to the next, and some devices are more flexible than others in the variants they can work with. 

The V4L2 layer represents video standards with the type `v4l2_std_id`, which is a 64-bit mask. Each standard variant is then one bit in the mask. So "standard" NTSC is `V4L2_STD_NTSC_M`, value `0x1000`, but the Japanese variant is `V4L2_STD_NTSC_M_JP` (`0x2000`). If a device can handle all variants of NTSC, it can set a standard type of `V4L2_STD_NTSC`, which has all of the relevant bits set. Similar sets of bits exist for the variants of PAL and SECAM. See [this page](<http://v4l2spec.bytesex.org/spec/r9288.htm#V4L2-STD-ID>) for a complete list. 

For user space, V4L2 provides an `ioctl()` command (`VIDIOC_ENUMSTD`) which allows an application to query which standards are implemented by a device. The driver does not need to answer those queries directly, however; instead, it simply sets the `tvnorm` field of the `video_device` structure with all of the standards that it supports. The V4L2 layer will then split out the supported standards for the application. The `VIDIOC_G_STD` command, used to query which standard is active at the moment, is also handled in the V4L2 layer by returning the value in the `current_norm` field of the `video_device` structure. The driver should, at startup, initialize `current_norm` to reflect reality; some applications will get confused if no standard is set, even though they have not set one. 

When an application wishes to request a specific standard, it will issue a `VIDIOC_S_STD` call, which is passed through to the driver via: 

```
int (*vidioc_s_std) (struct file *file, void *private_data,
                             v4l2_std_id std);
```

The driver should program the hardware to use the given standard and return zero (or a negative error code). The V4L2 layer will handle setting `current_norm` to the new value. 

The application may want to know what kind of signal the hardware actually sees on its input. The answer can be found with `VIDIOC_QUERYSTD`, which reaches the driver as: 

```
int (*vidioc_querystd) (struct file *file, void *private_data,
                                v4l2_std_id *std);
```

The driver should fill in this field in the greatest detail possible. If the hardware does not provide much information, the `std` field should indicate any of the standards which _might_ be present. 

There is one more point worth noting here: all video devices must support (or at least claim to support) at least one standard. Video standards make little sense for camera devices, which are not tied to any specific regulatory regime. But there is no standard for "I'm a camera and can do almost anything you want." So the V4L2 layer has a number of camera drivers which claim to return PAL or NTSC data. 

### Inputs

A video acquisition application will start by enumerating the available inputs with the `VIDIOC_ENUMINPUT` command. Within the V4L2 layer, that command will be turned into a call to the driver's corresponding callback: 

```
int (*vidioc_enum_input)(struct file *file, void *private_data,
    			     struct v4l2_input *input);
```

In this call, `file` corresponds to the open video device, and `private_data` is the private field set by the driver. The `input` structure is where the real information is passed; it has several fields of interest: 

  * `__u32 index`: the index number of the input the application is interested in; this is the only field which will be set by user space. Drivers should assign index numbers to inputs, starting at zero and going up from there. An application wanting to know about all available inputs will call `VIDIOC_ENUMINPUT` with index numbers starting at zero and incrementing from there; once the driver returns `EINVAL` the application knows that it has exhausted the list. Input number zero should exist for all input-capable devices. 

  * `__u8 name[32]`: the name of the input, as set by the driver. In simple cases, it can simply be "Camera" or some such; if the card has multiple inputs, the name used here should correspond to what is printed by the connector. 

  * `__u32 type`: the type of input. There are currently only two: `V4L2_INPUT_TYPE_TUNER` and `V4L2_INPUT_TYPE_CAMERA`. 

  * `__u32 audioset`: describes which audio inputs can be associated with this video input. Audio inputs are enumerated by index number just like video inputs (we'll get to audio in another installment), but not all combinations of audio and video can be selected. This field is a bitmask with a bit set for each audio input which works with the video input being enumerated. If no audio inputs are supported, or if only a single input can be selected, the driver can simply leave this field as zero. 

  * `__u32 tuner`: if this input is a tuner (`type` is set to `V4L2_INPUT_TYPE_TUNER`), this field will contain an index number corresponding to the tuner device. Enumeration and control of tuners will be covered in a future installment too. 

  * `v4l2_std_id std`: describes which video standard(s) are supported by the device. 

  * `__u32 status`: gives the status of the input. The [full set of flags](<http://v4l2spec.bytesex.org/spec/r8936.htm#INPUT-STATUS>) can be found in the V4L2 documentation; in short, each bit set in `status` describes a problem. These can include no power, no signal, no synchronization lock, or the presence of Macrovision, among other unfortunate events. 

  * `__u32 reserved[4]`: reserved fields. Drivers should set them to zero. 

Normally, the driver will set all of the fields above and return zero. If `index` is outside the range of supported inputs, `-EINVAL` should be returned instead; there is not much else that can go wrong in this call. 

When the application wants to change the current input, the driver will receive a call to its `vidioc_s_input()` callback: 

```
int (*vidioc_s_input) (struct file *file, void *private_data, 
                               unsigned int index);
```

The `index` value has the same meaning as before - it identifies which input is of interest. The driver should program the hardware to use that input and return zero. Other possible return values are `-EINVAL` (for a bogus index number) or `-EIO` (for hardware trouble). Drivers should implement this callback even if they only support a single input. 

There is also a callback to query which input is currently active: 

```
int (*vidioc_g_input) (struct file *file, void *private_data, 
                               unsigned int *index);
```

Here, the driver sets `*index` to the index number of the currently active input. 

### Outputs

The process for enumerating and selecting outputs is very similar to that for inputs, so the description here will be a little more brief. The callback for output enumeration looks like this: 

```
int (*vidioc_enumoutput) (struct file *file, void *private_data
        			      struct v4l2_output *output);
```

The fields of the `v4l2_output` structure are: 

  * `__u32 index`: the index value corresponding to the output. This index works the same way as the input index: it starts at zero and goes up from there. 

  * `__u8 name[32]`: the name of the output. 

  * `__u32 type`: the type of the output. The supported output types are `V4L2_OUTPUT_TYPE_MODULATOR` for an analog TV modulator, `V4L2_OUTPUT_TYPE_ANALOG` for basic analog video output, and `V4L2_OUTPUT_TYPE_ANALOGVGAOVERLAY` for analog VGA overlay devices. 

  * `__u32 audioset`: the set of audio outputs which can operate with this video output. 

  * `__u32 modulator`: the index of the modulator associated with this device (for those of type `V4L2_OUTPUT_TYPE_MODULATOR`). 

  * `v4l2_std_id std`: the video standards supported by this output. 

  * `__u32 reserved[4]`: reserved fields, should be set to zero. 

There are callbacks for getting and setting the current output setting; they mirror the input callbacks: 

```
int (*vidioc_g_output) (struct file *file, void *private_data, 
                                unsigned int *index);
        int (*vidioc_s_output) (struct file *file, void *private_data, 
                                unsigned int index);
```

Any device which supports video output should have all three output callbacks defined, even if there is only one possible output. 

With these methods in place, a V4L2 application can determine which inputs and outputs are available on a given device and choose between them. The task of determining just what kind of video data flows through those inputs and outputs is rather more complicated, however. The next installment in this series will begin to look at video data formats and how to negotiate a format with user space.
