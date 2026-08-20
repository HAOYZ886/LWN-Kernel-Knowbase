---
title: Hardware face detection
url: https://lwn.net/Articles/469662/
date: "November 29, 2011"
category: "Development model-User-space ABI; Device drivers-Video4Linux2"
author: "By Jonathan Corbet November 29, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 29, 2011

Once upon a time, a "system on chip" (SOC) was a package containing a processor and some number of I/O controllers. While SOCs still have all that, manufacturers have been busy adding hardware support for all kinds of interesting functionality. For example, OMAP4 processors have an onboard face detection module that can be used for camera focus control, "face unlock" features, and more. Naturally, there is interest in making use of such features in Linux; a recent driver submission shows that the question of just how to do that has not yet been answered, though. 

The [OMAP4 face ~~recognition~~ detection driver](<https://lwn.net/Articles/469647/>) was submitted by Tom Leiming, but was apparently written by Ming Lei. Upon initialization, the driver allocates a memory area which is made available to an application via `mmap()`. The application places an image in that area (it seems that a 320x240 grayscale PGM image is the only supported option), then uses a number of `ioctl()` operations to specify the area of interest and to start and stop the image recognition process. A [![\[face
recognition\]](https://static.lwn.net/images/2011/face-recog.png)](<http://kernel.ubuntu.com/~ming/dev/fdif/output/>) `read()` on the device will, once detection is complete, yield a number of structures describing the locations of the faces in the image as rectangles. 

Face detection functionality is clearly welcome, but this particular driver has a lot of problems and will not get into the mainline in anything resembling its current state. The most significant criticism, though, came from [Alan Cox](<https://lwn.net/Articles/469677/>), who asked that, rather than being implemented as a standalone device, face detection be integrated into the Video4Linux2 framework. 

In truth, V4L2 is probably the right place for this feature. Face detection is generally meant to be used with the camera controller integrated into the same SOC and the face detection hardware may be tightly tied to that controller. The [media controller](<https://lwn.net/Articles/415714/>) subsystem was designed for just this kind of functionality; it provides a mechanism by which camera data may (or may not) be routed to the face detection module as needed. Integration into V4L2 would bring the face detection module under the same umbrella as the rest of the video processing hardware and export the necessary data routing capabilities to user space. 

The design of the user-space interface for this functionality seems likely to pose challenges of its own, though. The OMAP4 hardware is relatively simple in its operation; it appears to even lack the ability to work with multiple image formats, even moderately high-resolution images, or color data. Future hardware will certainly not be so limited. It is also not hard to imagine a shift from detection of any face to recognition of specific faces - or, at least, the generation of metrics to ease the association of faces and the identities of their owners. The hardware could become capable of blink detection, distinguishing real faces from pictures of faces, or determining when a face belongs to a poker player who is bluffing. Designing an API that can handle this kind of functionality is going to be an interesting task. 

But it does not stop there. There is a discouragingly large market out there for devices capable of reading automobile license plates, for example. There is money in meeting the needs of the contemporary surveillance state, so manufacturers will certainly compete to provide the needed capabilities. In general, the world is filled with interesting things that are not faces; it is not hard to imagine that people will be able to do useful things with devices that can pick all kinds of high-level objects out of image data. 

In general, we may be seeing a shift in what kinds of peripherals are attached to our processors. There will always be plenty of devices that serve essentially (from the CPU's point of view) as channels moving chunks of data in one direction or the other. But there will be more and more devices that offload some type of processing, and that is going to present some interesting ABI challenges. Hardware-based offload engines are nothing new, of course. But, once upon a time, offload devices mostly performed tasks otherwise handled by the operating system kernel. Integrated controllers and network protocol offload functionality are a couple of obvious examples. More recently, though, hardware has provided functionality that needs to be made available to user space. And that changes the game somewhat. 

If one looks for examples of this kind of functionality, one almost certainly needs to start at the GPU found in most graphics cards. Creating a workable (and stable) user-space ABI providing access to the GPU has taken many years, and it is not clear that the job is done yet. The media controller ABI controls routing of data among the numerous interesting functional units in contemporary video processors, but writing a hardware-independent application using the media controller is hard. Creating a workable interface for the wide variety of available industrial sensors has also been a multi-year project. 

Trying to anticipate where this kind of hardware will go in an attempt to create the perfect ABI from the outset seems like an exercise in futility. Most likely it will have to be done the way we've always done it: come up with something that seems reasonable, learn (the hard way) what it's shortcomings are, then begin the long process of replacing it with something better. It is not an ideal way to create an operating system, but it seems to be better than the alternatives. Figuring out the best way to support face detection will just be another step in this ongoing process.
