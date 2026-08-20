---
title: From wakelocks to a real solution
url: https://lwn.net/Articles/319860/
date: "February 18, 2009"
category: "Android; Power management-Opportunistic suspend"
author: "By Jonathan Corbet February 18, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 18, 2009

[Last week's article on wakelocks](<http://lwn.net/Articles/318611/>) described a suspend-inhibiting interface which derives from the Android project and the hostile reaction that interface received. Since then, the discussion has continued in two separate threads. Kernel developers, like engineers everywhere, are problem solvers, so the discussion has shifted away from criticism of wakelocks and toward the search for an acceptable solution. As of this writing, that solution does not exist, but we have learned some interesting things about the problem space. 

Getting Linux power management to work well has been a long, drawn-out process, much of which involves fixing device drivers and applications, one at a time. There is also a lot of work which has gone into ensuring that the CPU remains in an idle state as much as possible. One of the reasons that some developers found the wakelock interface jarring was that the Android developers chose a different approach to power management. Rather than minimize power consumption at any given time, the Android code simply tries to suspend the entire device whenever possible. There are a couple of reasons for this approach, one of which we will get to below. 

But we'll start with a very simple reason why Android goes for the "suspend the entire world" solution: because they can. The hardware that Android runs on, like many embedded systems (but unlike most x86-based systems), has been designed to suspend and resume quickly. So the Android developers see no reason to do things any other way. But that leads to comments like [this one from Matthew Garrett](<https://lwn.net/Articles/319863/>): 

Part of the reason you're getting pushback is that your solution to the problem of shutting down unused hardware is tied to embedded-style systems with very low resume latencies. You can afford to handle the problem by entering an explicit suspend state. In the x86 mobile world, we don't have that option. It's simply too slow and disruptive to the user experience. As a consequence we're far more interested in hardware power management that doesn't require an explicit system-wide suspend. 

A solution that's focused on powering down as much unused hardware as possible regardless of the system state benefits the x86 world as well as the embedded world, so I think there's a fairly strong argument that it's a better solution than one requiring an explicit system state change. 

Matthew also notes that it's possible to solve the power management problem without fully suspending the system; he gives the Nokia tablets as an example of a successful implementation which uses finer-grained power management. 

That said, it seems clear that the full-suspend approach to power management is not going to go away. Some hardware is designed to work best that way, so Linux needs to support that mode of operation. So there has been some talk about how to design wakelocks in a way which fits better into the kernel as a whole. On the kernel side, there is some dispute as to whether the wakelock mechanism is needed at all; drivers can already inhibit an attempt by the kernel to suspend the system. But there is some justice to the claim that it's better if the kernel knows it can't suspend the system without having to poll every driver. 

One simple solution, [proposed by Matthew](<https://lwn.net/Articles/319869/>), would be a simple pair of functions: `inhibit_suspend()` and `uninhibit_suspend()`. On production systems, they would manipulate an atomic counter; when the counter is zero, the system can be suspended. These functions could take a `device` structure as an argument; debugging versions could then track which devices are blocking a suspend at any given time. The user-space equivalent could be a file like `/dev/inhibit_suspend`; as long as at least one process holds that file open, the system will continue to run. All told, it looks like a simple API without many of the problems seen in the wakelock code. 

There were a few complaints from the Android side, but the biggest sticking point appears to be over timeouts. The wakelock API implements an automatic timeout which causes the "lock" to go away after a given time. There appear to be a few reasons for the existence of the timeouts: 

  * Since not all drivers use the wakelock API, timeouts are required to prevent suspending the system while those drivers are running. The proposed solution to this one is to instrument all of the drivers which need to keep the system running. Once an acceptable API is merged into the kernel, drivers can be modified as needed. 

  * If a process holding a wakelock dies unexpectedly, the timeout will keep the system running while the watchdog code restarts the faulting process. The problem here is that timeouts encode a recovery policy in the kernel and do little to ensure that operation is actually correct. What has been proposed instead is that the user-space "inhibit suspend" policy be encapsulated into a separate daemon which would make the decisions on when to keep the system awake. 

  * User-space applications may simply screw up and forget to allow the system to suspend. 

The final case above is also used as an argument for the full-suspend approach to power management. Even if an ill-behaved application goes into a loop and refuses to quit, the system will eventually suspend and save its battery anyway. This is an argument which does not fly particularly well with a lot of kernel developers, who respond that, rather than coding the kernel to protect against poor applications, one should simply fix those applications. Arjan van de Ven [points out](<https://lwn.net/Articles/319874/>) that, since the advent of [PowerTop](<http://linuxpowertop.org/>), the bulk of the problems with open-source applications have been fixed. 

In this space, though, it is harder to get a handle on all of these problems. Brian Swetland [describes the situation](<https://lwn.net/Articles/319875/>) this way: 

  * carrier deploys a device 
  * carrier agrees to allow installation of arbitrary third party apps without some horrible certification program requiring app authors to jump through hoops, wait ages for approval, etc 
  * users rejoice and install all kinds of apps 
  * some apps are poorly written and impact battery life 
  * users complain to carrier about battery life 

Matthew also [acknowledges the problem](<https://lwn.net/Articles/319876/>): 

Remember that Android has an open marketplace designed to appeal to Java programmers - users are going to end up downloading code from there and then blaming the platform if their battery life heads towards zero. I think "We can't trust our userland not to be dumb" is a valid concern. 

It is a real problem, but it still is not at all clear that attempts to fix such problems in the kernel are advisable - or that they will be successful in the end. Ben Herrenschmidt [offers](<https://lwn.net/Articles/319877/>) a different solution: a daemon which monitors application behavior and warns the user when a given application is seen to be behaving badly. That would at least let users know where the real problem is. But it is, of course, no substitute for the real solution: run open-source applications on the phone so that poor behavior can be fixed by users if need be. 

The Android platform is explicitly designed to enable proprietary applications, though. It may prove to be able to attract those applications in a way which standard desktop Linux has never quite managed to do. So some sort of solution to the problem of power management in the face of badly-written applications will need to be found. The Android developers like wakelocks as that solution for now, but they also appear to be interested in working with the community to find a more globally-acceptable solution. What that solution will look like, though, is unlikely to become clear without a lot more discussion.
