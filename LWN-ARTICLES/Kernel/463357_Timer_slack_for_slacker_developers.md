---
title: Timer slack for slacker developers
url: https://lwn.net/Articles/463357/
date: "October 17, 2011"
category: Control groups; Timers
author: "By Jonathan Corbet October 17, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 17, 2011

The "timer slack controller" is a proposed mechanism that would allow a session management program to adjust the timer tolerances of a group of processes with a single knob. It seems like a relatively obscure and harmless feature, but it has been the focus of an intense debate on the kernel mailing lists. The core question has been seen before: what measures should the kernel take, if any, to keep poorly-written applications from hurting performance? 

Timers allow a process to request a wakeup at some future time; timer slack gives the kernel some leeway in its implementation of those timers. If the kernel can delay specific timers by a bounded amount, it can often expire multiple timers at once, minimizing the number of wakeups and, thus, reducing the system's power consumption. Some processes need more precise timing than others; for this reason, the kernel allows a process to specify its maximum timer slack with the `prctl()` system call. There is, currently, no mechanism to allow one process to adjust another process's timer slack value; it is generally assumed that any given process knows best when it comes to its own timing requirements. 

The [timer slack controller](<https://lwn.net/Articles/462566/>) allows a suitably privileged process to set the timer slack value for every process contained within a control group. The patch has been circulating for some time without generating a great deal of interest; it recently resurfaced in response to the "[plumber's wish list for Linux](<https://lwn.net/Articles/462076/>)" which requested such a feature. The reasoning behind the request was [explained](<https://lwn.net/Articles/463367/>) by Lennart Poettering: 

Consider you have one or more desktop user sessions logged in, each one in a timer slack cgroup. Now, userspace already tracks when sessions become idle (i.e. currently desktop userspace then starts a screensaver, or turns off the screen, or similar), and we'd like to increase the timer slack for the session cgroups individually as the individual session becomes idle, and decrease it again if the session stops being idle. 

It is, in other words, a power-saving mechanism. When the session manager determines that nothing special is going on, it can massively increase the slack on any timers operated by desktop applications, effectively decreasing the number of wakeups. Applications need not be aware of whether the user is currently at the keyboard or not; they will simply slow down during the boring times. 

There is some stiff opposition to merging this controller. Naturally, the fact that the timer slack controller uses control groups is part of the problem; some kernel developers have still not made their peace with control groups. Until that situation resolves itself - if it ever does - features based on control groups are going to have a bumpy ride on their way into the mainline. 

Beyond the general control group issue, though, two complaints have been heard about this approach to power management. One is that applications running on the desktop may have timing requirements that are not dependent on whether the user is actually there or not. One could imagine a data acquisition application that does not have stringent response requirements, but which will still lose data if its timers suddenly gain multiple seconds of slack. Lennart's [response](<https://lwn.net/Articles/463374/>) is that such applications should be using the realtime scheduler classes, but that answer is unlikely to please anybody. There is likely to be no shortage of applications that have never needed to bother with realtime scheduling but which still will not work well with arbitrary delays. Imposing such delays could lead to any number of strange bugs. 

The big complaint, though, as [expressed](<https://lwn.net/Articles/463376/>) by Peter Zijlstra and others, is that this feature makes it easier for developers to get away with writing low-quality applications. If the pressure to remove badly-written code is removed, it is said, that code will never get fixed. Peter suggests that, rather than papering over poor behavior in the kernel, it would be better to simply kill applications that waste power. He was especially strident about applications that continue to draw when their windows are not visible; such problems should be fixed, he said, before adding workarounds to the kernel. 

The massive improvements in power behavior that resulted from the release and use of PowerTop is often pointed to as an example of how things should be done. This situation is a little different, though. The wakeup reductions inspired by PowerTop were low-hanging fruit - processes waking up multiple times per second for no useful purpose. The timer slack controller is aimed at a different problem: wakeups which _are_ useful when somebody is paying attention, but which are not useful otherwise. That is a trickier problem. 

Determining when the user is paying attention is not always straightforward, though there some obvious signs. If the screen has been turned off because the input devices are idle, the user probably does not care. Other cases - non-visible tabs in web browsers, for example - have been cited as well, but the situation is not so obvious there. As Matthew Garrett [put it](<https://lwn.net/Articles/463390/>): buried tabs still need timer events ""because people expect gmail to provide them with status updates even if it's not the foreground tab."" Fixing the problem in applications would require figuring out when nothing is going on, finding a way to communicate it to applications, then fixing large numbers of them (some of which are proprietary) to respond to those events. 

It is not surprising that developers facing that kind of challenge might choose to improve the situation with a simple kernel patch instead. It is, certainly, a relatively easy path toward better battery life. But the patch does raise a fundamental policy question that has never been answered in any definitive way. Does mitigating the effects of (what is seen as) application developer sloppiness encourage the distribution of low-quality code and worsen the system in the long run? Or, instead, does the "tough love" approach deter developers and impoverish our application environment without actually fixing the underlying problems? 

An answer to that question is unlikely to come in the near future. What that probably means is that the current fuss will be enough to keep the timer slack controller from getting in through the 3.2 merge window. It also seems unlikely to go away, though; we are likely to see this topic return to the mailing lists in the future.
