---
title: Waking systems from suspend
url: https://lwn.net/Articles/429925/
date: "March 2, 2011"
category: Timers
author: "March 2, 2011 This article was contributed by John Stultz"
---

March 2, 2011

This article was contributed by John Stultz

While the power consumption of an idle Linux system has been reduced greatly over the past few years, even more power can be saved by suspending or hibernating the system. Resume times have also gone down, increasing the usability of suspending a laptop even if you're just walking down the hallway to a meeting. And while suspend and hibernation were once features only found on portable devices like laptops, they have over the years become common on mobile embedded devices and non-portable desktops and servers. The power-saving benefits of suspend and hibernate come from the fact that most or all of the hardware is shut down, but this can be a limitation if you're expecting some functionality out of the system. It's the same reason sleeping at your desk is usually frowned upon. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

But let's just say, if you were an extraordinary cat-napper, and you had some downtime between numerous kernel compiles while doing a long git-bisect: You could make it work, but first you would need a good alarm clock. The same can be said of computers. 

#### The RTC

The RTC (Real Time Clock) is a fairly minor bit of hardware on your computer. It usually keeps track of the wall-clock time while the system is off or suspended. It also can be used to generate interrupts in a number of different modes (periodic, one-shot alarm, etc). This is all fairly normal functionality for a hardware timer device. But one of the most interesting features that most modern RTCs support is that an alarm interrupt can be generated even when the system is suspended (or in some hardware hibernation) forcing the machine to wake up. 

On Linux the RTC is exposed to user space via the generic RTC driver infrastructure, which creates sysfs entries and a character device which can be used to set hardware alarms, change the interrupt mode, etc. A few applications out there make use of this interface, such as [MythTV DVRs](<http://www.mythtv.org/>), which can trigger alarms so that media computers can be suspended until the start of a TV show that needs to be recorded. 

The exposed interface is very much a low-level driver interface, where the values written by the application are sent directly to the hardware. This is a limitation, as it makes it so only one application at a time can program alarm events to an RTC device. For instance, with only a single RTC device, you can't have your system wake up for a nightly backup and also have it wake up to record your favorite show, unless you have some sort of centralized process managing the wakeups on behalf of other applications. Tutorials such as [this one](<http://www.osnews.com/story/24111/Wake_Your_Linux_Up_From_Sleep_for_a_Cron_Job>) illustrate how complex and limiting this interface can be. 

One way to overcome these limitations is to allow the kernel to manage a list of events and have it program the RTC so the alarm will trigger for the earliest event in the list. This avoids the need for user space applications to coordinate in order to share the hardware. To make this sharing possible, a generic "timerqueue" abstraction has been created to manage a simple list of timers that could then be shared with other areas of the kernel, like the high-resolution timers subsystem, that also have to manage timer events. This code was merged for 2.6.38. 

The next step is to rework the RTC code so that, when an alarm is set via the character device `ioctl()` or sysfs interface, an `rtc_timer` event is created and enqueued into the per-RTC timerqueue instead of directly programming the hardware. The kernel then sets the hardware timer to fire for the earliest event in the queue. In effect, this mechanism virtualizes the RTC hardware, preserving the behavior of the existing hardware-oriented interfaces, while allowing the kernel to multiplex other events using the RTC. 

The question now becomes, how to expose this new functionality so it can be used? 

#### CLOCK_RTC

The first approach tried was exporting the new RTC functionality to user space directly via the POSIX clocks and timers interface. With this approach, there is a "clockid" assigned to each RTC device, so a user space application can use the POSIX interfaces to access the RTC. In this approach, `clock_gettime()` returns the current RTC time, `clock_settime()` sets the RTC time, and `timer_settime()` sets a POSIX timer to expire when the RTC reaches the desired time. 

This approach is the most straightforward method of exposing the RTC, but it does have some disadvantages. Specifically, the RTC and system time may not be the same. On many systems, the RTC is set to local time rather than universal time. Thus, applications would need to make the extra effort to read the RTC and add to that value the time between now and when they want the timer to fire. Also, the RTC, due to simple clock skew, may not increase at the exact same rate as the system time. Additionally, since there may be multiple RTCs on a system, a single static `CLOCK_RTC` clockid would not be sufficient. Some form of dynamic `clock_id` registration is needed in order to export multiple clockids for multiple RTC devices. This functionality is desired for exposing other hardware clocks via the POSIX interface, and it is currently a [work-in-progress by Richard Cochran.](<http://lwn.net/Articles/413332/>)

#### Android Alarm Timers

Interestingly, the developers who have been working on Android have extended the RTC to be more useful as well. After all, smartphones are optimized to save power, so they try to stay in suspend as much as possible. But smartphones still have to wake up to do things like notify the user of calendar events or to check for email. In order to do this, The Android team introduced a concept called [Android Alarm Timers.](<http://android.git.kernel.org/?p=kernel/common.git;a=blob;f=drivers/rtc/alarm.c;hb=HEAD>) These timers use a hybrid approach: when when the system is running, alarm timers trigger a high-res timer to fire when an event is supposed to run; however, when the system goes into suspend, the alarm timers code looks at the list of events and sets the RTC to fire an alarm when the earliest event is to run. This avoids making applications deal with the (possibly unsyncronized) RTC time domain and allows applications to simply set timers and have them fire when expected, whether or not the system is suspended. 

While never submitted to the kernel mailing list for inclusion, the Android Alarm Timers implementation would likely meet some resistance from the kernel community. For instance, the user-space interface for applications to use the Android Alarm Timers is via `ioctl()` to a new special character device (`/dev/alarm`) instead of using existing system call interfaces. Additionally, the `ioctl()` interface introduces new names for existing concepts in the kernel, duplicating CLOCK_REALTIME (which provides UTC wall time) and CLOCK_MONOTONIC (which counts from zero starting at system boot, and is not modified by `settimeofday()` calls) via the names ANDROID_ALARM_RTC and ANDROID_ALARM_SYSTEMTIME respectively. 

The Android Alarm Timers interface does introduce some new useful concepts. For instance, the CLOCK_MONOTONIC clock does not increment during suspend. This is reasonable behavior when you want suspend to be transparent to applications, but when the system spends the majority of its time in suspend and you want to schedule events that wake the system up having only CLOCK_REALTIME increment over suspend can be limiting. So Android Alarm Timers introduces the ANDROID_ALARM_ELAPSED_REALTIME clock, which is similar to CLOCK_MONOTONIC, but includes time spent in suspend. But again, it is only introduced via an `ioctl()` to their special character device, and is not exposed via any other standard timekeeping interface. 

#### Posix Alarm Timers

All in all, the Android Alarm Timers are a very interesting use case, and others in the community have suggested a similar hybrid approach. Inspired by the Android Alarm Timers, I implemented a similar hybrid alarm timers infrastructure on top of the previously-described work virtualizing the RTC interface. However, these timers are exposed to user space via the standard POSIX clocks and timers interface, using the new the CLOCK_REALTIME_ALARM clockid. The new clockid behaves identically to CLOCK_REALTIME except that timers set against the _ALARM clockid will wake the system if it is suspended. Additionally, because it's built upon the virtualized `rtc_timers` work, this implementation doesn't prohibit applications from making use of the existing legacy RTC interfaces. This gives us all the benefits of Android Alarm Timers, such as not forcing applications to deal with the RTC time domain, while making better use of existing kernel interfaces. 

The code that implements the timerqueues and reworks the generic RTC layer to allow for multiplexing of events has been included in the 2.6.38 kernel release. The POSIX alarm timers layer will likely need additional review and discussion, in hopes of making sure the Android developers are able to assess compatibility issues in the design. For instance, I've proposed a new POSIX clock (CLOCK_BOOTTIME, along with a corresponding CLOCK_BOOTTIME_ALARM id) which would provide the incrementing-in-suspend value that the Android developers introduced with ANDROID_ALARM_ELAPSED_REALTIME. Also, while not likely to be included into mainline, Android's wakelocks have some interesting semantics with regards to their alarm timer interface. These semantics are not easily satisfied by the posix timers interface, but it is to be determined if we can get equivalent functionality using modified semantics and the mainline kernel's [pm_wakeup interface.](<http://lwn.net/Articles/416690/>)

Other open questions that need to be addressed are: 

  * What capabilities should applications be required to have in order to set POSIX alarm timers? 

  * In order to avoid systems waking up at inappropriate times (think laptop in a bag in the overhead compartment), should there be additional policy layers added so that user-generated suspends (like closing a laptop) inhibit POSIX alarm timers? 

I also can imagine some interesting future work combining this functionality with the "Wake on Directed Packet" feature of some new network cards, which wake the system up any time a packet is sent to it. This feature could be used to allow web servers to function normally, servicing requests and running jobs, while suspending and saving power during longer idle periods. 

While I might not be able to sleep on the job, I look forward to my desktop system being able to snooze and save electricity while knowing that cron jobs like nightly backups, downloading package updates or running updatedb will still be done.
