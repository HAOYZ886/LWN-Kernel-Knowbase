---
title: In search of the right RGB LED interface
url: https://lwn.net/Articles/687496/
date: "May 25, 2016"
category: "Device drivers-Support APIs"
author: "May 25, 2016 This article was contributed by Neil Brown"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

May 25, 2016

This article was contributed by Neil Brown

One of the roles of the Linux kernel is to provide uniform, abstract interfaces to varying hardware. When a new class of hardware comes along, it can take a while to understand what the best interface would be. This has been seen in recent months with the appearance of nonvolatile memory in large quantities leading to disagreements over the semantics of DAX filesystem access and the handling of hardware errors. The same basic question has arisen, though in a much smaller way, over the best handling of RGB LEDs — triplets of LEDs, each of a different color, which together can produce a wide range of colors and intensities.

Linux already has support for monochrome LEDs, including minimal support for identifying the color of each LED: the name of the LED can, and sometimes does, include the English name of the LED's color (`locomo:green:mail`, for example). The simplest approach to managing RGB LEDs is to treat them as three independent LEDs with related names. User-space tools can then follow simple conventions to find related LEDs and create interesting colors as required.

There are two reasons to think this may not be the best long-term solution. The first involves integration with the various "triggers" that Linux supports for LEDs. As Jacek Anaszewski from Samsung [explains](<http://mid.gmane.org/5714A502.9030201@samsung.com>), there are two classes of source information for triggers. One class has the trigger local to the LED, such as "timer" or "[oneshot](<https://lwn.net/Articles/508810/>)". These triggers are controlled from user space; programming three triggers in concert might be a little clumsy, but it still allows the full functionality to be used.

The other class of source information is from in-kernel events: CPU load, disk drive activity, network device activity, etc. These currently only adjust the brightness or the duty-cycle of the LEDs, but a natural enhancement would be to allow them to adjust color. That would require the kernel to know how specific LEDs work together to produce different colors. A particular example is the heartbeat trigger. On monochrome LEDs this trigger produces a "[thump-thump-pause](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/leds/trigger/ledtrig-heartbeat.c?id=b562e44f507e863c6792946e4e1b1449fbbac85d#n44>)" pattern designed to mimic the human heartbeat with a rate that increases as the load-average on the system increases. Heiner Kallweit has [implemented](<http://article.gmane.org/gmane.linux.leds/4716>) an alternate heartbeat that works with RGB LEDs and uses the color (ranging from green to red) rather than the rate to represent load.

It is easy to imagine other ways that color information could be used to represent such things as acceptable or worrisome activity from various parts of the kernel. Supporting direct connections from those subsystems to a suitable RGB LED may provide a lot of value.

The second reason that the kernel might benefit from an explicit understanding that three LEDs work together is that this understanding is embedded in some hardware. A good example is the [LP5523 LED controller [PDF]](<http://www.ti.com/lit/ds/symlink/lp5523.pdf>) from TI that can drive up to nine LEDs. This controller is programmable with three separate engines and space to store 96 16-bit instructions. The instructions are general enough to be usable for [computing prime numbers](<http://pavelmachek.livejournal.com/120188.html>). The three engines naturally align with three sets of RGB LEDs, so allowing the kernel interface to represent these triples is likely to make for a better interface. Even when the LEDs are only accessed from user space, it would be helpful if high-level program requests, such as blink rates or brightness transitions, could be described for the three together so they can reliably be synchronized.

As yet there does not seem to be a clear vision for how generic RGB support might work. Kallweit posted [some patches](<http://thread.gmane.org/56D608ED.2090406@gmail.com>) back in March but they have some problems. The basic approach is to present the three LEDs as a single LED device that changes all three colors at the same time, so it can be used as though it were a single white LED. The "brightness" value can be given hue and saturation components as well; this allows color to be changed from user space. This triplet of values is encoded in a single sysfs attribute which, as Pavel Machek [highlighted](<http://mid.gmane.org/20160329100258.GA24964@amd>), is not generally seen as acceptable.

One argument against this approach is that there are already devices with tri-color LEDs, such as the Nokia N900 and the motion controller for the Sony PlayStation. These currently use three separate LED devices and they need to be able to continue to work the same way when new functionality is added.

Using HSV (Hue Saturation Value) has some appeal as it includes the current brightness as a subset but, for correct mapping to RGB, a "gamma" value needs to be included, and the kernel may not be the best place to be adding that sort of complexity.

After some discussion, Anaszewski came up with a [proposal](<http://mid.gmane.org/5714A502.9030201@samsung.com>) that could make triggers like the color-based load indicator work with individual red, green, and blue LEDs. A single trigger can already apply to multiple LEDs, so the first step would be to assign that colorful "heartbeat" to each of the three LEDs. Then a new sysfs attribute would be used to configure each one to only display the "red", "green", or "blue" component of the signal. While this feels a little clumsy, it would certainly work and is simple to implement and to understand, which are more important considerations.

This doesn't really address the need to be able to program controllers that expect LEDs to be related rather than completely independent. Machek has [some ideas](<http://mid.gmane.org/20160401135748.GD11860@amd>) on how that might be approached. There isn't a lot of detail; it essentially involves creating a new "pattern" device in sysfs that represents the capabilities for the engine in the controller. It can be configured and then linked to one or more LED devices. This model seems flexible enough to be able to support both software and hardware pattern generators, but without more details (and code) it is hard to judge it fairly.

Keeping the individual LEDs separate, but allowing them to be combined for pattern generation, seems to be a fairly accurate model of how the hardware works, as there is nothing in the hardware controller that forces the LEDs to be mounted close to each other physically. This match between model and reality bodes well for the design being one that could be successful.

While little details like RGB LEDs might not get as much attention as big-ticket items like massive nonvolatile RAM arrays, they are still quite important as they are exactly the sort of thing we can expect to see more of in the mobile-device space. If we ever want these devices to run mainline kernels, we would do well to work on getting support of these devices into mainline first. Or at least a close second.
