---
title: Likely unlikely()s
url: https://lwn.net/Articles/420019/
date: "December 15, 2010"
category: likely
author: "By Jake Edge December 15, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
December 15, 2010

The kernel has two macros to assist the compiler and CPU in doing branch prediction: `likely()` and `unlikely()`. As their names imply, they are meant to annotate tests in the code based on the likelihood that they will evaluate to true. But, getting it wrong such that something marked likely is actually unlikely--or vice versa--can impact performance as the CPU may prefetch the wrong instructions. 

Steven Rostedt has been [looking at the problem](<https://lwn.net/Articles/419102/>) using the annotated branch profiler and found ten places ""that really do not need to have an annotated branch, or the annotation is simply the opposite of what it should be"". So, he created a series of patches that either switched the sense of the annotation or removed the `likely()`/`unlikely()` entirely. 

As an example, `page_mapping()` had an `unlikely()` annotation that Rostedt [reported](<https://lwn.net/Articles/420028/>) as being ""correct a total of 1909540379 times and incorrect 1270533123 times, with a 39% being incorrect"". Those numbers come from his main workstation which runs a variety of standard programs (Firefox, XChat, etc.) as well as participating in his build farm, so it should represent a reasonably "normal" workload. Being wrong 39% of the time was pretty obviously too much and led to the removal of the annotation for that test. 

The changes are various subsystems including the scheduler, memory management, and VFS. So far, there have been no complaints, though there have been several requests to completely remove annotations that had just been changed in order to allow the compiler's and CPU's branch prediction logic make the decision. That, and breaking the patches up into separate sets for each subsystem, caused Rostedt to respin them. It would seem `likely()` that we'll see them make their way into 2.6.38.
