---
title: Character devices in user space
url: https://lwn.net/Articles/308445/
date: "November 25, 2008"
category: "Device drivers-In user space"
author: "By Jake Edge November 25, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 25, 2008

There is a lot of functionality--things like filesystems and device drivers--that are normally considered to be kernel tasks, but have, over time, been allowed to move into user space. The [UIO user space driver framework](<http://lwn.net/Articles/232575/>) came along in 2.6.23, while filesystems in user space (FUSE) have been around since 2.6.14. Tejun Heo would like to see this idea broadened even further with the [character devices in user space](<http://lwn.net/Articles/308112/>) (CUSE) patches. 

At first blush, the uses for a character device implemented in user space are not obvious. Looking a bit deeper, though, one finds numerous programs--both open and closed source--that rely on legacy character drivers. Those drivers are currently in the kernel, but need not be if there were a way to implement them in user space. In addition, older, deprecated interfaces, such as Open Sound System (OSS) can be better supported without constantly fiddling with the in-kernel emulation. 

Providing better OSS support is one of the prime motivators for CUSE as Heo announced in a linux-kernel [posting](<https://lwn.net/Articles/308490/>) introducing the [OSS proxy](<http://userweb.kernel.org/~tj/ossp/>). The proxy uses CUSE to implement the `/dev/dsp`, `/dev/adsp`, and `/dev/mixer` devices that programs using OSS expect. Adrian Bunk didn't necessarily [see this as a good thing](<https://lwn.net/Articles/308525/>): 

Sorry for being destructive, but 6 years after ALSA went into the kernel we are slightly approaching the point where all applications support ALSA. 

The application you list on your webpage is UML host sound support, and I'm wondering why you don't fix that instead of working on a better OSS emulation? 

But Heo sees the current state of OSS emulation as a rather complicated mess that, for better or worse, [needs cleaning up](<https://lwn.net/Articles/308531/>): 

We now have in-kernel OSS emulation which can't mux with other streams, aoss [ALSA OSS emulation] with its own supported and broken list and can also be routed through PA [PulseAudio] by configuring ALSA right and then padsp [PA OSS emulation] with its own supported and broken list and nothing works good enough. So, if we have one thing which just works, we can in time put all those to rest. 

But there are other uses for CUSE too. Greg Kroah-Hartman [notes](<https://lwn.net/Articles/308550/>) that legacy software for talking to Palm Pilots, much of which is binary-only, expects to talk to a `/dev/pilot` serial port. The kernel carries around a driver, but ""a libusb userspace program can handle all of the data to the USB device instead"". So CUSE could be used to eventually remove another crufty driver from the kernel, while still maintaining compatibility with old user space code. 

CUSE is implemented on top of FUSE as there is a fair amount of overlap between them. Character devices and filesystems implement many of the same file operations--things like `open()`, `close()`, `read()`, and `write()`--which makes them a good match. Heo has a separate [patchset for FUSE](<http://lwn.net/Articles/308111/>) that implements additional operations for filesystems some of which will be used by CUSE. 

The additional FUSE operations include an implementation of `ioctl()` that is necessarily rather ugly. Because an `ioctl` implementation can access memory in unpredictable ways--and those data structures can be arbitrarily deep--there needs to be a mechanism for user-space CUSE devices to read and write that memory. The CUSE server does not have direct access to the caller's memory, so a multi-step `ioctl()` with retries must be implemented. This particular bit of ugliness is only allowed for in-kernel use, so that CUSE (or other things like it) can allow "unrestricted" `ioctl()` implementations. All FUSE filesystems are still required to have "restricted" `ioctls` where the kernel can determine the direction and amount of data that is transferred. `poll()` support has also been added to FUSE, which, in turn, requires a separate patch that allows `poll()` callbacks to sleep (described in this [article](<http://lwn.net/Articles/308426/>)). 

Once the FUSE changes are in place, the actual implementation of CUSE is relatively small, weighing in around 1000 lines plus some housekeeping to rename and export FUSE symbols. At its core, it collects up a FUSE-mounted filesystem that connects to the user-space implemented device along with the kernel-exported character device, binding the two together. FUSE handles the interaction with the user-space code, in the same way that it does for a filesystem. 

CUSE creates a device for commands, `/dev/cuse`, which is opened by a program that wants to implement a particular character device. CUSE queries the opener to determine which device it is implementing and then creates the device node. For most operations, CUSE just hands off to FUSE, but for `open()` it, instead, opens a file from the FUSE mount, storing the file handle for use by later operations. 

In many ways, CUSE is a kind of impedance matching layer that creates something that acts like a character device, but has no hardware directly behind it. This allows CUSE to ignore things like hardware interrupts; those would need to be handled by something else, typically a downstream driver--the soundcard driver in the OSS proxy case. This is one of the big differences between UIO and CUSE. UIO is much more like a regular kernel device driver that requires kernel code to handle interrupts. CUSE drivers, on the other hand, can be created without ever touching kernel space. 

The only objection so far seems to be Bunk's complaint about supporting OSS when it has been deprecated for so long. As Heo points out, though, there are still many applications that only support OSS. In addition, all of the code that has been submitted is ""way smaller than the in-kernel ALSA OSS emulation which is somewhat painful to use these days"", [Heo says](<https://lwn.net/Articles/308590/>). Since there are other potential users of CUSE, not just the OSS proxy, it would seem that, absent any major objections, CUSE could make it into 2.6.29.
