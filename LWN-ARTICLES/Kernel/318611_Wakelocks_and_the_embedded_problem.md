---
title: Wakelocks and the embedded problem
url: https://lwn.net/Articles/318611/
date: "February 10, 2009"
category: Android; Embedded systems
author: "By Jonathan Corbet February 10, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 10, 2009

The relationship between embedded system developers and the kernel community is known for being rough, at best. Kernel developers complain about low-quality work and a lack of contributions from the embedded side; the embedded developers, when they say anything at all, express frustrations that the kernel development process does not really keep their needs in mind. A current discussion involving developers from the Android project gives some insight into where this disconnect comes from. 

Android, of course, is Google's platform for mobile telephones. The initial Android stack was developed behind closed doors; the code only made it out into the world when the first deployments were already in the works. The Android developers have done a lot of kernel work, but very little code has made made the journey into the mainline. The code which _has_ been merged all went into the staging tree without a whole lot of initiative from the Android side. Now, though, Android developer Arve Hjønnevag is making an effort to merge a piece of that project's infrastructure through the normal process. It is not proving to be an easy ride. 

The most controversial bit of code is a feature known as "wakelocks." In Android-speak, a "wakelock" is a mechanism which can prevent the system from going into a low-power state. In brief, kernel code can set up a wakelock with something like this: 

```
#include <linux/wakelock.h>
    
        wake_lock_init(struct wakelock *lock, int type, const char *name);
```

The `type` value describes what kind of wakelock this is; `name` gives it a name which can be seen in `/proc/wakelocks`. There are two possibilities for the type: `WAKE_LOCK_SUSPEND` prevents the system from suspending, while `WAKE_LOCK_IDLE` prevents going into a low-power idle state which may increase response times. The API for acquiring and releasing these locks is: 

```
void wake_lock(struct wake_lock *lock);
        void wake_lock_timeout(struct wake_lock *lock, long timeout);
        void wake_unlock(struct wake_lock *lock);
```

There is also a user-space interface. Writing a name to `/sys/power/wake_lock` establishes a lock with that name, which can then be written to `/sys/power/wake_unlock` to release the lock. The [current patch set](<http://lwn.net/Articles/318588/>) only allows suspend locks to be taken from user space. 

This submission has not been received particularly well. It has, instead, drawn comments like [this from Ben Herrenschmidt](<https://lwn.net/Articles/318612/>): 

looks to me like some people hacked up some ad-hoc trick for their own local need without instead trying to figure out how to fit things with the existing infrastructure (or possibly propose changes to the existing infrastructure to fit their needs). 

or [this one from Pavel Machek](<https://lwn.net/Articles/318613/>): 

Ok, I think that this wakelock stuff is in "can't be used properly" area on Rusty's scale of nasty interfaces. 

There's no end of reasons to dislike this interface. Much of it duplicates the existing pm_qos (quality of service) API; it seems that [pm_qos does not meet Android's needs](<https://lwn.net/Articles/318614/>), but it also seems that no effort was made to fix the problems. The scheme seems over-engineered when all that is really needed is a "do not suspend" flag - or, at most, a counter. The patches disable the existing `/sys/power/state` interface, which does not play well with wakelocks. There is no way to recover if a user-space process exits while holding a wakelock. The default behavior for the system is to suspend, even if a process is running; keeping a system awake may involve a chain of wakelocks obtained by various software components. And so on. 

The end result is that this code will not make it into the mainline kernel. But it has been shipped on large numbers of G1 phones, with many more yet to go. So users of all those phones will be using out-of-tree code which will not be merged, at least not in anything like its current form. Any applications which depend on the wakelock sysfs interface will break if that interface is brought up to proper standards. It's a bit of a mess, but it is a very typical mess for the embedded systems community. Embedded developers operate under a set of constraints which makes proper kernel development hard. For example: 

  * One of the core rules of kernel development is "post early and often." Code which is developed behind closed doors gets no feedback from the development community, so it can easily follow a blind path for a long time. But embedded system vendors rarely want to let the world know about what they are doing before the product is ready to ship; they hope, instead, to keep their competitors in the dark for as long as possible. So posting early is rarely seen as an option. 

  * Another fundamental rule is "upstream first": code goes into the mainline before being shipped to customers. Once again, even if an embedded vendor wants to send code into the mainline, they rarely want to begin that process before the product ships. So embedded kernels are shipped containing out-of-tree code which almost certainly has a number of problems, unsupportable APIs, and more. 

  * Kernel developers are expected to work with the goal of improving the kernel for everybody. Embedded developers, instead, are generally solving a highly-specific problem under tight time constraints. So they do not think about, for example, extending the existing quality-of-service API to meet their needs; instead, they bash out something which is quick, dirty, and not subject to upstream review. 

One could argue that Google has the time, resources, and in-house kernel development knowledge to avoid all of these problems and do things right. Instead, we have been treated to a fairly classic example of how things can go wrong. 

The good news is that Google developers are now engaging with the community and trying to get their code into the mainline. This process could well be long, and require a fair amount of adjustment on the Android side. Even if the idea of wakelocks as a way to prevent the system from suspending is accepted - which is far from certain - the interface will require significant changes. The associated "early suspend" API - essentially a notification mechanism for system state changes - will need to be generalized beyond the specific needs of the G1 phone. It could well be a lot of work. 

But if that work gets done, the kernel will be much better placed to handle the power-management needs of handheld devices. That, in turn, can only benefit anybody else working on embedded Linux deployments. And, crucially, it will help the Android developers as they port their code to other devices with differing needs. As the number of Android-based phones grows, the cost of carrying out-of-tree code to support each of them will also grow. It would be far better to generalize that support and get it into the mainline, where it can be maintained and improved by the community. 

Most embedded systems vendors, it seems, would be unwilling to do that work; they are too busy trying to put together their next product. So this sort of code tends to languish out of the mainline, and the quality of embedded Linux suffers accordingly. Perhaps this case will be different, though; maybe Google will put the resources into getting its specialized code into shape and merged into the mainline. That effort could help to establish Android as a solid, well-supported platform for mobile use, and that should be good for business. Your editor, ever the optimist, hopes that things will work out this way; it would be a good demonstration of how embedded community can work better with the kernel community, getting a better kernel in return.
