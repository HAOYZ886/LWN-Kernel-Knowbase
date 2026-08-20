---
title: Short sleeps suffering from slack
url: https://lwn.net/Articles/482168/
date: "February 17, 2012"
category: "Development model-User-space ABI; Timers; hrtimer"
author: "By Jonathan Corbet February 17, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 17, 2012

As a general rule, kernel developers will go out of their way to avoid breaking user-space code, even when that code is seen as being wrong and already broken. But there are exceptions; a recent discussion regarding timer behavior may prove to be an example of how such exceptions can come about. 

The C-library `sleep()` function is defined to put the calling process to sleep for at least the number of seconds specified. One might think that calling `sleep()` with an argument of zero seconds would make relatively little sense; why put a process to sleep for no time? It turns out, though, that some developers put such calls in as a way to relinquish the CPU for a short period of time. The idea is to be nice and allow other processes to run briefly before continuing execution. Applications that perform polling or are otherwise prone to consuming too much CPU are often "fixed" with a zero-second sleep. 

Once upon a time in Linux, `sleep(0)` would always put the calling process to sleep for at least one clock tick. When high-resolution timers were added to the kernel, the behavior changed: if a process asked to sleep on an already-expired timer (which is the case for a zero-second sleep), the call simply returned directly back to the calling process. Then came the addition of timer slack, which can extend sleep periods to force multiple processes to wake at the same time. This behavior will cause timers to run a little longer than requested, but the result is fewer processor wakeups and, thus, a savings of power. In the case of a zero-second sleep, the addition of timer slack turns an expired timer into one that is not expired, so the calling process will, once again, be put to sleep. 

The default timer slack, at 50µs, is unlikely to cause visible changes to the behavior of most applications. But it seems that, on some systems, the timer slack value is set quite high - on the order of seconds - to get the best power behavior possible. That can extend the length of a zero-second sleep accordingly, leading to misbehaving applications. 

Matthew Garrett, working under the notion that breaking applications is bad, submitted [a patch](<https://lwn.net/Articles/482178/>) making a special-case for zero-second sleeps. The idea is simple: if the requested sleep time is zero, timer slack will not be added and the process will not be delayed indefinitely. The problem with this approach is that the process will still not get the desired result: rather than yielding the processor, it will have simply performed a useless system call and gone right back to whatever it was doing before. Without timer slack, a request to sleep on an expired timer will return directly to user space without going through the scheduler at all. 

An alternative would be to transform `sleep(0)` into a call to `sched_yield()`. But that idea is not hugely popular with the scheduler developers, who think that calls to `sched_yield()` are almost always a bad idea. It is better, they say, to fix the applications to stop polling or doing whatever else it is that they do that causes developers to think that explicitly yielding the CPU is the right thing to do. 

[According to Matthew](<https://lwn.net/Articles/482180/>), the number of affected applications is not tiny: 

Checking through an exploded Fedora kernel tree suggests around 125 packages out of 11000 or so, so around 1% of userspace seems to use sleep(0) under certain circumstances. We can probably fix everything in the distribution, but that suggests that there's also going to be a significant amount of code in the outside world that's also broken. 

Normal practice in kernel development would be to try to avoid breaking those applications if possible. Even in cases where applications are relying on undefined and undocumented behavior - certainly the case here - it is better if a kernel upgrade doesn't turn working code into broken code. Some participants have suggested that the same approach should be taken in this case. 

The situation with `sleep(0)` is a little different from others, though. Application developers cannot claim a long history of working behavior in this case, since the kernel's response to a zero-second sleep has already changed a few times over the course of the last decade. And, [according to Thomas Gleixner](<https://lwn.net/Articles/482197/>), it is hard to know when the special case applies or what should be done: 

Dammit, we cannot come up with a reasonable definition for special casing that stuff simply because you cannot draw a clear boundary what to special case and what not. And there is no sensible definition for what to do - return right away or go through schedule() or what ever. 

Thomas worries that there may be calls for special cases for similar calls \- single-nanosecond calls to `nanosleep()`, for example - and that the result will be an accumulation of cruft in the core timer code. So, rather than try to define these cases and maintain the result indefinitely, he thinks it is better just to let the affected code break in cases where the timer slack has been set to a large value. And that is where the discussion faded away, suggesting that nothing will be done in the kernel to reduce the effect of timer slack on zero-second sleeps.
