---
title: "Leap-second issues, 2015 edition"
url: https://lwn.net/Articles/648313/
date: "June 17, 2015"
category: Timekeeping
author: "By Jonathan Corbet June 17, 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 17, 2015

The leap second is an occasional ritual wherein Coordinated Universal Time (UTC) is held back for one second to account for the slowing of the Earth's rotation. The last leap second happened on June 30, 2012; the next is scheduled for June 30 of this year. Leap seconds are thus infrequent events. One might easily imagine that infrequent events involving time discontinuities would be likely to expose software problems, and, sure enough, the 2012 leap second [had its share of issues](<https://lwn.net/Articles/504744/>). The 2015 leap second looks to be a calmer affair, but it appears that it will not be entirely problem-free. 

Prarit Bhargava [reported](<https://lkml.org/lkml/2015/5/27/458>) a problem at the end of May: it seems that, when the leap second hits, some timers can fire one second earlier than they should. This is not a good outcome; timers can be delayed, but they should never fire before their appointed time. It did not take long to understand the problem, but finding a proper solution was a rather slower task. 

Linux handles leap seconds in the kernel. An application (typically the network time protocol (NTP) daemon) informs the kernel of an upcoming leap second via the [`adjtimex()` system call](<http://man7.org/linux/man-pages/man2/adjtimex.2.html>); when the appointed time arrives, the system clock will be set back by one second. There is an important detail in all of this, though: this adjustment happens during the normal timer tick. The tick is not precisely lined up with the second boundaries as determined by UTC, so there is a window of time between the beginning of the leap second and when the kernel figures out that it needs to hold the clock back. The window is brief (a maximum of about 10ms, usually shorter), but that's enough time for timers set for just after midnight to fire. 

One might argue that one second every few years is not a big problem, and that applications that really care should be using the [International Atomic Time](<https://en.wikipedia.org/wiki/International_Atomic_Time>) (TAI) clock anyway. There _are_ applications with precise timekeeping requirements; some of them are certainly using the UTC clock rather than TAI, but they should work anyway if possible. So this seems like a problem worth fixing. 

John Stultz's [first attempt](<https://lwn.net/Articles/648319/>) at providing that fix did not go entirely well, though. In particular, the [patch](<https://lwn.net/Articles/648320/>) that moved the leap-second adjustments into the timer fast-path code ran into opposition. The patch added some significant complexity to code that is already far from simple, and it threatened to slow down some of the most frequently exercised code in the kernel. The [loudest opposition](<https://lwn.net/Articles/648324/>) came from Ingo Molnar, who asked: ""why do we add over 100 lines of code for something that occurs (literally) once in a blue moon?"" 

Ingo's suggestion was to implement [leap-second smearing](<http://googleblog.blogspot.com/2011/09/time-technology-and-leaping-seconds.html>) instead. The smearing approach does away with the time discontinuity by tweaking the speed of the clock instead; it would handle a leap second insertion by running the clock just a little slower for a number of hours prior to the event. There are no abrupt time transitions, and no weird times (like 23:59:60) that applications may not be prepared to deal with. It could, Ingo [said](<https://lwn.net/Articles/648325/>), also be handled almost entirely from user space via `adjtimex()`, allowing administrators to control policy and getting the kernel out of the leap-second business entirely. 

In truth, life is not so simple. Clocks that do not have leap seconds (the TAI clock in particular) should not smear, so the kernel would have to stay involved and it would have to maintain time bases running at different speeds while the smearing was happening. As John [noted](<https://lwn.net/Articles/648327/>), that does not look like a path leading to lower levels of complexity. He argued that, while smearing looks like a worthwhile thing to add to the kernel, it does not address the immediate problem. 

Ingo also [argued](<https://lwn.net/Articles/648328/>) that the leap-second code should not support leap-second deletion, where the clock is set _ahead_ by one second. Deletion has never happened, and, when one looks at the physical processes involved in the slowing of the Earth's rotation, it seems like it probably never will (""discounting massive asteroid strikes, at which point leap seconds will be the least of our problems""). Other operating systems are unlikely to handle deletion gracefully; if they support it at all, it is with code that has never been exercised in the real world. The `adjtimex()` interface allows for it, though, so John argues that it should be supported; the code is already there and seems unlikely to be removed. 

After the discussion calmed down, John came back with [a reduced patch set](<https://lwn.net/Articles/648329/>) limited to the essential fix. In this version of the patch, the timer code does the leap-second adjustment early enough to avoid premature timer expirations; one might suggest that the new timer code looks before it leaps. The rest of the code remains untouched, though, so the performance impact of the change is minimal. This version found a friendlier reception; it has been added to the "tip" tree for merging into the 4.2 kernel. 

As of this writing, the leap-second insertion is less than two weeks away, so John's fix is clearly not destined to be deployed worldwide before that happens. But getting it into the mainline now will ensure that it's running on at least some development systems when the leap second hits, giving it a certain amount of real-world testing. That should increase confidence in the correctness of the patch and help to ensure that, when the next leap second is declared, the kernel will handle it properly.
