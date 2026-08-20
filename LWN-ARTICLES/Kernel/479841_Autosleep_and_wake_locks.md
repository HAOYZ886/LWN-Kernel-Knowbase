---
title: Autosleep and wake locks
url: https://lwn.net/Articles/479841/
date: "February 7, 2012"
category: "Android; Power management-Opportunistic suspend"
author: "By Jonathan Corbet February 7, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 7, 2012

The [announcement](<https://lwn.net/Articles/472984/>) of the Android merging project and the return of a number of Android-specific drivers to the kernel's staging tree were notable events in December, 2011. The most controversial Android change - "wakelocks" or "suspend blockers" - is not a part of this effort, though. That code is sufficiently intrusive and sufficiently controversial that nobody seemed to want to revisit it at this time. Except that, as it turns out, one person is still trying to make progress on this difficult issue. Rafael Wysocki's [autosleep and wake locks patch set](<https://lwn.net/Articles/479711/>) is yet another attempt to support Android's opportunistic suspend mechanism in a mainline kernel. 

"Opportunistic suspend" is a heavy-handed approach to power management. In short, whenever nothing of interest is going on, the entire system simply suspends itself. It is certainly effective on Android devices; in particular, it prevents poorly-written applications from keeping the system awake and draining the battery. The hard part is the determination that nothing interesting is happening; that is the role of the Android wakelock/suspend blocker mechanism. With suspend blockers, both the kernel and suitably-privileged user-space code are able to block the normal suspension of the system, keeping it running for whatever important work is being done. 

Given that suspend blockers do not seem to be headed into the mainline kernel anytime soon, some sort of alternative mechanism is required if the mainline is to support opportunistic suspend. As it happens, some pieces of that solution have been in the mainline for a while; the [wakeup events](<https://lwn.net/Articles/416690/>) infrastructure was merged for 2.6.36. Wakeup events track events (a button press, for example) that can wake the system or keep it awake. "Wakeup sources," which track sources of wakeup events, were merged for 2.6.37. Thus far, the wakeup events subsystem remains lightly used in the kernel; few drivers actually signal such events. Wakeup sources are almost entirely unused. 

Rafael's patch set makes some significant changes that employ this infrastructure to support "autosleep," which is another word for "opportunistic suspend." (Rafael says: ""This series tests the theory that the easiest way to sell a once rejected feature is to advertise it under a different name""). The first of those adds a new file to sysfs called `/sys/power/autosleep`; writing "`mem`" to this file will cause the system to suspend whenever there are no active wakeup sources. One can also write "`disk`", with the result that the system will opportunistically hibernate; this feature may see rather less real-world use, but it was an easy addition to make. 

The Android system tracks the time that suspend blockers prevent the system from suspending; that information is then used in the "why is my battery dead?" screen. Rafael's patch adds a similar tracking feature and exports this time (as `prevent_sleep_time`) in `/sys/kernel/debug/wakeup_sources`. 

One little problem remains, though: wakeup sources are good for tracking kernel-originated events, but they do not provide any way for user space to indicate that the system should not sleep. What's needed, clearly, is a mechanism with which user space can create its own wakeup sources. The final patch in Rafael's series adds just such a feature. An application can write a name (and an optional timeout) to `/sys/power/wake_lock` to establish a new, active wakeup source. That source will prevent system suspend until either its timeout expires or the same name is written to `/sys/power/wake_unlock`. 

It is easy to see that this mechanism can be used to implement Android's race-free opportunistic suspend. A driver receiving a wakeup event will mark the associated wakeup source as active, keeping the system running. That source will stay active until user space has consumed the event. But, before doing so, the user-space application takes a "wake lock" of its own, ensuring that it will be able to complete its processing before the system goes back to sleep. 

Those who have been paying attention to this controversy will have noted that the API for this feature looks suspiciously like the native Android API. Needless to say, that is not a coincidence; the idea is to make it as easy as possible to switch over to the new mechanism without breaking Android systems. If that goal can be achieved, then, even if Android itself never moves to this implementation, it should be that much easier to run an Android user space on a mainline kernel. 

And that, of course, will be the ultimate proof of this patch set. If somebody is able to demonstrate an Android system running with native opportunistic suspend, with similar power consumption characteristics, then it's a lot more likely that this patch will succeed where so many others have failed. Arranging such a demonstration will not be entirely easy, but, on the right hardware, it is certainly possible. Linaro's [Android build for the Pandaboard](<https://wiki.linaro.org/Platform/Android/BuildSource>) might be a good starting point. Until that happens, getting an Android-compatible opportunistic suspend implementation into the mainline could be challenging.
