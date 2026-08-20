---
title: "The Video4Linux2 API: an introduction"
url: https://lwn.net/Articles/203924/
date: "October 11, 2006"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: 
---

Your editor has recently had the opportunity to write a Linux driver for a camera device - the camera which will be packaged with the One Laptop Per Child system, in particular. This driver works with the internal kernel API designed for such purposes: the Video4Linux2 API. In the process of writing this code, your editor made the shocking discovery that, in fact, this API is not particularly well documented - though the user-space side is, instead, [quite well documented indeed](<http://v4l2spec.bytesex.org/spec/book1.htm>). In an attempt to remedy the situation somewhat, LWN will, over the coming months, publish a series of articles describing how to write drivers for the V4L2 interface. 

V4L2 has a long history - the first gleam came into Bill Dirks's eye back around August of 1998. Development proceeded for years, and the V4L2 API was finally merged into the mainline in November, 2002, when [2.5.46](<http://lwn.net/Articles/14568/>) was released. To this day, however, quite a few Linux drivers do not support the newer API; the conversion process is an ongoing task. Meanwhile, the V4L2 API continues to evolve, with some major changes being made in 2.6.18. Applications which work with V4L2 remain relatively scarce. 

V4L2 is designed to support a wide variety of devices, only some of which are truly "video" in nature: 

  * The **video capture interface** grabs video data from a tuner or camera device. For many, video capture will be the primary application for V4L2. Since your editor's experience is strongest in this area, this series will tend to emphasize the capture API, but there is more to V4L2 than that. 

  * The **video output interface** allows applications to drive peripherals which can provide video images - perhaps in the form of a television signal - outside of the computer. 

  * A variant of the capture interface can be found in the **video overlay interface** , whose job is to facilitate the direct display of video data from a capture device. Video data moves directly from the capture device to the display, without passing through the system's CPU. 

  * The **VBI interfaces** provide access to data transmitted during the video blanking interval. There are two of them, the "raw" and "sliced" interfaces, which differ in the amount of processing of the VBI data performed in hardware. 

  * The **radio interface** provides access to audio streams from AM and FM tuner devices. 

Other types of devices are possible. The V4L2 API has some stubs for "codec" and "effect" devices, both of which perform transformations on video data streams. Those areas have not yet been completely specified, however, much less implemented. There are also the "teletext" and "radio data system" interfaces currently implemented in the older V4L1 API; those have not been moved to V4L2 and there do not appear to be any immediate plans to do so. 

Video devices differ from many others in the vast number of ways in which they can be configured. As a result, much of a V4L2 driver implements code which enables applications to discover a given device's capabilities and to configure that device to operate in the desired manner. The V4L2 API defines several dozen callbacks for the configuration of parameters like tuner frequencies, windowing and cropping, frame rates, video compression, image parameters (brightness, contrast, ...), video standards, video formats, etc. Much of this series will be devoted to looking at how this configuration process happens. 

Then, there is the small task of actually performing I/O at video rates in an efficient manner. The V4L2 API defines three different ways of moving video data between user space and the peripheral, some of which can be on the complex side. Separate articles will look at video I/O and the video-buf layer which has been provided to handle common tasks. 

Subsequent articles will appear every few weeks, and will be added to the list below: 

  * [Part 2: registration and `open()`](<https://lwn.net/Articles/204545/>)
  * [Part 3: Basic ioctl() handling](<https://lwn.net/Articles/206765/>)
  * [Part 4: Inputs and Outputs](<http://lwn.net/Articles/213798/>)
  * [Part 5a: Colors and formats](<http://lwn.net/Articles/218798/>)
  * [Part 5b: Format negotiation](<https://lwn.net/Articles/227533/>)
  * [Part 6a: Basic frame I/O](<https://lwn.net/Articles/235023/>)
  * [Part 6b: Streaming I/O](<http://lwn.net/Articles/240667/>)
  * [Part 7: Controls](<http://lwn.net/Articles/247126/>)
