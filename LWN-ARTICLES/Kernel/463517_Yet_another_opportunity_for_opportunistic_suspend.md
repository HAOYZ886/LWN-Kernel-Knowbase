---
title: Yet another opportunity for opportunistic suspend
url: https://lwn.net/Articles/463517/
date: "October 18, 2011"
category: "Android; Power management-Opportunistic suspend"
author: "By Jonathan Corbet October 18, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 18, 2011

Your editor was innocently looking at some papers on his desk the other day when his computer abruptly decided to suspend itself. Rawhide is fun in that way; combined with GNOME's delight in forgetting user settings, it can produce no end of surprises to brighten one's working experience. The ability to suspend a desktop system to RAM is actually quite a nice feature, but your editor prefers to have a say in when it happens. Thankfully, GNOME still (so far) allows automatic suspend to be turned off. But it is clear that the suspend-to-RAM functionality is seeing increased use in a number of contexts; it is not just for laptops and Android anymore. Your editor's desktop is not the only place where stakeholders want some control over when the system sleeps and when it needs to stay running. 

Indeed, control over automatic suspension of the system is at the core of the debate over Android's opportunistic suspend mechanism. As usage of suspend-to-RAM increases, so does interest in creating a proper mechanism for determining when a suspend can happen. [A new patch set](<https://lwn.net/Articles/463336/>) from Rafael Wysocki has restarted this discussion and led to, possibly, a surprising conclusion. 

Rafael started with the conclusion that ""whatever the kernel has to offer in this area is either too complicated to use in practice or inadequate for other reasons"". He then went on to propose a new mechanism that, he hoped, would simplify things. It came in two parts: 

  * A new sysfs knob, `/sys/power/sleep_mode`, which provided overall control of the suspend-to-RAM and hibernation functionality. If a suitably-privileged process writes "`disabled`" to this file, no attempt to suspend or hibernate the system will succeed. It is a sort of high-power wakelock that ensures the system will keep running while important work is being done. 

  * Applications wanting to keep the system awake would open a new device, `/dev/sleepctl`, and execute an `ioctl()` to that effect. After this call, attempts to suspend the system would block until the application explicitly drops its lock or until a 500ms (by default) timeout period expires. The "stay awake" operation would also be done by the system at resume time to give processes time to perform whatever tasks need to be done. 

It is probably safe to say that these patches will not be merged in anything resembling this form. Leading the opposition was Neil Brown, who [asserted](<https://lwn.net/Articles/463530/>) that the job could be done in user space, and, indeed, should be done that way. According to Neil: 

The only sane way to handle suspend is for any (suitably privileged) process to be able to request that suspend doesn't happen, and then for one process to initiate suspend when no-one is blocking it. 

Communication with that process, Neil claimed, should be no harder than using Rafael's simplified interface to communicate with the kernel. After a fair amount of discussion, Neil came up with [a proposal](<https://lwn.net/Articles/463510/>) for how he thinks things should actually work. As one would expect from the above quote, it centers around a single daemon with the responsibility for suspending and resuming the system. A decision to suspend the system is never made by the kernel, and, if everybody is following the rules, by no other user-space process. 

The daemon has a pair of modes; it starts in the "on demand" mode where the system will only be suspended after an explicit request to do so. That request could come from the user closing the lid or pressing a button sequence; in this case, the system should suspend in short order regardless of what is happening, and it should not resume without an explicit user action. Suspend can also be requested by a suitably-privileged application; in this case the operation is only carried out if nothing is blocking it, and the system can be automatically resumed at some future time. This mode was also referred to as the "legacy" mode; it needs to be supported but it is not how things are expected to run most of the time. 

Other processes in the system can affect suspend behavior by talking to the daemon. One of the things a sufficiently-privileged process can do is to ask the daemon to go into "immediate" mode; in that mode, the system will suspend anytime there is no known reason to stay awake. The immediate mode, thus, closely mirrors the opportunistic suspend mechanism used on Android systems. When the daemon is in immediate mode, it no longer makes sense for any process in the system to ask the system to suspend - the daemon is already prepared to suspend whenever the opportunity arises. So the rest of the interface is concerned with when the system should be awake. 

Any process with an interest in suspend and resume events can open a socket to the daemon and request notification anytime a suspend is being contemplated. That process should respond to such notifications with a message saying that it is ready for the suspend to happen; it can, optionally, add a request that the system stay awake for the time being if there is work that must be done. If no processes block the suspend, the system will go to sleep; another message will be sent to all processes once the system resumes. 

There is an interesting variant on this mechanism whereby processes can register one or more file descriptors with the daemon. In this case, the daemon will only query the associated processes before suspending if one or more of the given file descriptors is reported as readable by `poll()`. A readable file descriptor thus functions in a manner similar to a driver-acquired wakelock in the Android system. If a device wakes the system and provides input for a user-space process to read, the daemon will see that the file descriptor is readable and avoid suspending the system until that input has been consumed and acted upon. Meanwhile, processes that clearly have no need to block suspend will not need to wake up and respond to a notification every time a suspend is contemplated. 

The daemon also allows processes to request that the system be awake at some future time. A tool like `cron` can use this feature to, say, wake the system late at night to run a backup. 

At a first glance, this approach looks like it should be able to handle the opportunistic suspend problem without the need to add more mechanism to the kernel. But it must be remembered that this is a problem that has defeated a number of initially reasonable-looking solutions. Whether this proposal will fare better - and whether the various desktop and mobile environments will adopt it - remains to be seen.
