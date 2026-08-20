---
title: The ACPI processor aggregator driver
url: https://lwn.net/Articles/355934/
date: "October 7, 2009"
category: ACPI
author: 
---

Patches merged into the mainline carry a number of tags to indicate who wrote them, who reviewed them, etc. A [certain commit](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=8e0af5141ab950b78b3ebbfaded5439dcf8b3a8d>) merged for 2.6.32 contains a relatively unusual tag, though: 

```
NACKed-by: Peter Zijlstra <a.p.zijlstra@chello.nl>
```

The merging of this patch has drawn some complaints: why should it have made it into the mainline when a core developer clearly has problems with it? 

The story goes something like this. ACPI provides a mechanism by which it can ask the system to make processors go idle in emergency situations; these can include power problems or an overheating system. The ACPI folks had originally proposed putting some hacks into the scheduler to implement this functionality. These changes, it seems, were little loved; _that_ was the patch that Peter Zijlstra blocked outright. 

So Shaohua Li went back and implemented this functionality as a driver instead. If the ACPI hardware starts sounding the red alert, this driver will create a top-priority realtime thread and bind it to the CPU that is to be idled. That thread, when it "runs," will simply put the CPU into a relatively deep sleep state for a while. When the emergency passes, the thread will go away and normal life resumes. It's a bit of a hack, but it gets the job done, and it is not destructive to system state the way hot-unplugging the CPU would be. 

The proper fix would be to enhance the scheduler (the right way) to provide this functionality. But that almost certainly requires the intervention of a real scheduler hacker, and they haven't yet gotten around to solving the problem. So the ACPI "driver" is in the mainline for now. And it may stay that way; Linus [said](<https://lwn.net/Articles/355937/>): 

In fact, the only reason the scheduler people even know about it is that Len at first tried to do something more invasive, and was shot down. Now it's just a driver, and the scheduler people can _try_ to do it some other way if they really care, but that's _their_ problem. Not the driver. 

In the meantime, I personally suspect we probably never want to even try to solve it in the scheduler, because why the hell should we care and add complex logic for something like that? At least not until we end up having the same issue on some other architecture too, and it turns from a hacky ACPI thing into something more. 

And that's where things stand. The driver is little loved, but it will also be little used, can be replaced with a better mechanism if the right people care, and, in the mean time, it may solve a real problem for some users.
