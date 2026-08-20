---
title: Suspend block
url: https://lwn.net/Articles/385103/
date: "April 28, 2010"
category: "Android; Power management-Opportunistic suspend"
author: "By Jonathan Corbet April 28, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 28, 2010

One of the key points of contention when it comes to getting the kernel-level Android code merged has been the [wakelock](<http://lwn.net/Articles/319860/>) API. Wakelocks run counter to how some think power management should be done, so they have been hard to merge. But Android's drivers use wakelocks, so, in the absence of that API, those drivers also cannot be merged. They could be reworked to not use wakelocks, but then the mainline kernel would have a forked version of the driver code which nobody actually uses - not the best of outcomes. So coming to resolution on the wakelock issue has been a high priority for a while. 

The result of work in that area can now be seen in the form of the [suspend block patches](<http://lwn.net/Articles/385051/>) recently posted by Arve Hjønnevåg. The name of the feature has been changed, as has the API, but the core point is the same: allow the system to automatically suspend itself when nothing is going on, and allow code to say "something is going on" at both the kernel and user-space levels. 

The suspend block patches add a new sysfs file called `/sys/power/policy`; the default value found therein is "forced." When the policy is "forced," system state transitions will happen in response to explicit writes to `/sys/power/state`, as usual. If the policy is changed to "opportunistic," though, things are a bit different. The state written to `/sys/power/state` does not take effect immediately; instead, the kernel goes into that state whenever it concludes that the system is idle. The suspend blocker API can then be used to prevent the system from suspending when the need arises. 

The two postings of this patch set have received a number of comments, causing various things to be fixed. More recently, though, responses have been of the "acked" variety. So one might conclude that suspend block has a reasonable chance of getting in during the 2.6.35 merge window. That, in turn, should open the doors for the merging of a lot of driver code from the Android project. With luck, the much-publicized disconnect between Android and the community may be a thing of the past - at the kernel level, at least.
