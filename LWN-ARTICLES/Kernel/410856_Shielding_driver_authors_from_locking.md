---
title: Shielding driver authors from locking
url: https://lwn.net/Articles/410856/
date: "October 20, 2010"
category: Big kernel lock; Device drivers
author: "By Jonathan Corbet October 20, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 20, 2010

Much of the time, patches can be developed against the mainline kernel and submitted for the next merge window without trouble. At other times, though, the mainline is far removed from the environment a patch will have to fit into at merge time. Your editor, who has been trying the (considerable) patience of the Video4Linux maintainer by trying to get a driver merged for 2.6.37 at the last minute, has encountered this fact of life the hard way: he submitted a driver which did not even come close to compiling inside the 2.6.37 V4L2 tree. Things have changed considerably there. This article will look at one of those changes with an eye toward the kind of design decisions that are being made in that part of the kernel. 

The removal of the big kernel lock (BKL) has been documented here over the years. One of the biggest holdouts at this point is the V4L2 subsystem; almost everything that happens in a V4L2 driver is the result of an `ioctl()` call, and those calls have always been protected by the BKL. Removing BKL protection means auditing the drivers - and there are a lot of them - and, in many cases, providing a replacement locking scheme. It seems that a lot of V4L2 drivers - especially the older ones - do not exhibit the sort of attention to locking that one would expect from code submitted today. 

The approach to this problem chosen by the V4L2 developers has proved to be mildly controversial within the group: they have tried to make it possible for driver authors to continue to avoid paying attention to locking. To that end, the `video_device` structure has gained a new `lock` field; it is a pointer to a mutex. If that field is non-null, the V4L2 core will acquire the mutex before calling any of the vast number of driver callbacks. So all driver operations are inherently serialized and driver authors need not worry about things. At least, they need not worry in the absence of other types of concurrency - like interrupts. 

Hans Verkuil, the developer behind many recent V4L2 improvements, clearly [feels](<https://lwn.net/Articles/410858/>) that it's better to handle the locking centrally: 

If he wants to punish himself and do all the locking manually (and prove that it is correct), then by all means, do so. If you want to use the core locking support and so simplify your driver and allow your brain to concentrate on getting the hardware to work, rather than trying to get the locking right, then that's fine as well. As a code reviewer I'd definitely prefer the latter approach as it makes my life much easier. 

On the other side, developers like Laurent Pinchart [argue](<https://lwn.net/Articles/410859/>) that trying to insulate developers from locking is not the right approach: 

Developers must not get told to be stupid and don't care about locks just because other developers got it wrong in the past. If people don't get locking right we need to educate them, not encourage them to understand even less of it. 

Central locking at the V4L2 level leads to some interesting problems as well. The V4L2 user-space [streaming API](<http://lwn.net/Articles/240667/>) offers a pair of `ioctl()` calls for the management of frame buffers: `VIDIOC_DQBUF` to obtain a buffer from the driver, and `VIDIOC_QBUF` to give a buffer back. If there are no buffers available at the time of the call, `VIDIOC_DQBUF` will normally block until a buffer becomes available. When this call is protected by the BKL, blocking will automatically release the lock and enable other V4L2 operations to continue. That behavior is important: one of those other operations might be a `VIDIOC_QBUF` call providing the buffer needed to allow the `VIDIOC_DQBUF` call to proceed; if `VIDIOC_DQBUF` fails to release the lock, things could deadlock. 

Drivers which handle their own locking will naturally release locks before blocking in a situation like this. Either the driver author thinks of it at the outset, or the need is made clear by disgruntled users later on. If the driver author is not even aware that the lock exists, though, it's less likely that the lock will be released at a time like this. That could lead to surprises in drivers which do their own I/O buffer management. If, however, the driver uses [videobuf](<http://lwn.net/Articles/363349/>), this problem will be handled transparently with some scary-looking code in `videobuf_waiton()`: 

```
is_ext_locked = q->ext_lock && mutex_is_locked(q->ext_lock);
    
        /* Release vdev lock to prevent this wait from blocking outside access to
           the device. */
        if (is_ext_locked)
    	mutex_unlock(q->ext_lock);
```

With enough due care, one assumes that it's possible to be sure that unlocking a mutex acquired elsewhere is a reasonable thing to do here. But one must hope that the driver author - who is not concerned with locking, after all - has left things in a consistent state before calling `videobuf_waiton()`. Otherwise those disgruntled users will eventually make a return. 

Locking complexity in the kernel is growing, and that makes it harder for developers to come up to speed. Complex locking can be an especially difficult challenge for somebody writing this type of driver; these authors tend not to be full-time kernel developers. So the appeal of taking care of locking for them and letting them concentrate on getting their hardware to do reasonable things is clear, especially if it makes the code review process easier as well. Such efforts may ultimately be successful, but there can be no doubt that they will run into disagreement from those who feel that kernel developers should either understand what is going on or switch to Java development. Expect this sort of discussion to pop up in a number of contexts as core developers try to make it easier for others to write correct code.
