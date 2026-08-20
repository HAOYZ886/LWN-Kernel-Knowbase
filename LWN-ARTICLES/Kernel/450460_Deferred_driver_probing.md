---
title: Deferred driver probing
url: https://lwn.net/Articles/450460/
date: "July 7, 2011"
category: Device drivers
author: "By Jonathan Corbet July 7, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 7, 2011

The developers working on the initial OLPC laptop ran into an interesting problem: the camera driver would fail to initialize if it was built into the kernel, but it worked just fine if built as a module. That problem still exists; it is a symptom of an issue which comes up frequently in contemporary systems: there is no way to know at build time what dependencies exist between different hardware units, so there is no way to ensure that drivers are loaded in the right order. A new patch from Grant Likely tries to solve that problem in a simple sort of way; it will probably improve the situation, but a complete solution is still lacking. 

The problem with the camera driver is a result of the fact that the "camera" is, in reality, three devices working in concert: a DMA bridge, a sensor, and an I2C bus connecting the two. The bridge (which plays the role of the overall "camera driver") must locate and identify the sensor as part of its setup routine; if the sensor does not exist, initialization will fail. But the sensor will not exist until its driver and the I2C bus driver have been loaded into the system. If all of the drivers are built into the kernel, the bridge driver's `probe()` function may be called first; there will be no sensor, so everything fails. 

Contemporary systems - especially those of the mobile variety - are increasingly built this way. Grant gave [another example](<https://lwn.net/Articles/450463/>): 

A "sound card" typically consists of multiple devices; one or more codecs (often i2c or spi attached), a sound bus (often i2s), a dma controller, and a lump of machine/platform specific code that ties them all together. Right now the ASoC code is going through all kinds of gymnastics make each component register with the ASoC layer and the 'tie together' driver has to wait for each of them to show up. 

The key point to understand is that the various components that make up a "device" may appear to be entirely unrelated at the hardware level. They can be on different buses; some of them may be subcomponents of entirely different devices. A general-purpose kernel has no real way to know what the real dependencies between devices are until all of the pieces are present and have started to recognize each other. 

Grant's [patch](<https://lwn.net/Articles/450178/>) takes a simple approach to solving this problem: drivers which are unable to initialize their devices as the result of missing resources can request that the operation be retried at some point in the future. That request is a simple matter of returning `-EAGAIN` from the `probe()` function. The driver core maintains a simple linked list of drivers that have requested this sort of deferral; when the time seems right, the deferred `probe()` invocations are retried to see if things work any better. 

One of the concerns raised with regard to this patch had to do with the determination of the right time. How might the kernel know when a failed initialization might work? The event which may change the situation is the successful addition of a new device to the system, so the current patch retries all of the deferred calls every time a new device shows up. The mechanism used for the retries (a workqueue) will tend to coalesce these attempts when a lot of devices are being registered (during system boot, for example), but it still strikes some reviewers as being inefficient. Grant has promised a revision of the patch which improves the situation. 

A related question is: when can the kernel conclude that there is no longer any point in retrying a specific driver's `probe()` function? In today's dynamic hardware environment, there never comes a point where one can say that no more devices will show up. This question has no real answer; it could be that, in a poorly configured or broken system, the process will never terminate. The cost of a driver stuck in the deferred state should be small, though. 

Others have questioned the need for this mechanism at all, but the responses have made it clear that something needs to be done to address this kind of hardware. A proper solution in the driver core seems like a better answer than a bunch of one-off hacks in specific drivers. So something will probably go in. 

Someday perhaps we will see a more elegant and efficient mechanism. One could imagine an API allowing a driver to specify which resources it is looking for; that driver's `probe()` function would then be put on hold until those resources become available. The driver core already generates events when new devices become available; some code matching those events to waiting drivers could be the last piece. But there would be a need to come up with a language by which a driver could express a need like "a device at address 42 on this I2C bus"; getting that right could take some work. 

Meanwhile, Grant's patch offers a "good enough" solution which appears capable of solving the problem most of the time. Accepting "good enough" when it's truly good enough is a key part of pragmatic programming. So chances are we'll have deferred driver initialization in the kernel sometime soon; fancier mechanisms may be rather longer in coming.
