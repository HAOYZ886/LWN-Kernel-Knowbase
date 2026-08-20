---
title: "Realtime group scheduling doesn't know JACK"
url: https://lwn.net/Articles/420407/
date: "December 19, 2010"
category: "Latency; Realtime; Scheduler-Group scheduling"
author: "By Jonathan Corbet December 19, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 19, 2010

Realtime scheduling for audio applications (or the lack thereof) has been a recurring theme over a number of years. LWN last [visited this issue](<https://lwn.net/Articles/339316/>) in 2009, when the addition of rtkit was put forward as the (pulseaudio-based) solution for casual audio use. Serious audio users - those using frameworks like [JACK](<http://jackaudio.org/>) \- have always wanted more direct access to realtime scheduling, though. That access has, for some years, been provided through resource limits. Now it seems that a feature merged for the 2.6.25 kernel is, two years later, beginning to cause grief for some JACK users. The resulting discussion is an interesting illustration of technical differences, how long it can take for new features to filter through to users, and how one should best deal with the kernel development community. 

The combination of the RLIMIT_RTPRIO and RLIMIT_RTTIME resource limits allows the system administrator to give specific users the ability to run tasks with realtime priority for a bounded period of time. The feature is easily configured in `/etc/security/limits.conf` and will prevent casual users from locking up the system with a runaway realtime process. This feature is limited in its flexibility, though, and is relatively easy to circumvent, so it has never been seen as an ideal solution. 

The better way, from the point of view of the scheduler developers, is to use realtime group scheduling. Group scheduling uses control groups to isolate groups of processes from each other and to limit the degree to which they can interfere with each other; there has been an increase in interest in group scheduling recently because this feature can be used to improve interactivity on loaded systems. But group scheduling can also be used to give limited access to realtime scheduling in a way which cannot be circumvented and which guarantees that the system cannot be locked up by a rogue process. It is a flexible mechanism which can be configured to implement any number of policies - even if the full feature set has not yet been implemented. More information on how this feature works can be found in [sched-rt-group.txt](<https://lwn.net/Articles/420408/>) in the kernel documentation tree. 

If realtime group scheduling is enabled in the kernel configuration, access to realtime priority based on resource limits is subordinated to the limits placed on the control group containing any given process. So if a process is run in a control group with no access to realtime scheduling, that process will not be able to put itself into a realtime scheduling class regardless of any resource limit settings. And that is where the trouble starts. 

The kernel, by default, grants realtime access to the "root" control group \- the one which contains all processes in the absence of some policy to the contrary. So, with a default setup, processes will be able to use resource limits to run with realtime priority. If, however, (1) the [libcgroup](<http://libcg.sourceforge.net/>) package has been installed, and (2) that package has been configured to put all user processes into a default (non-root) group, the situation changes. The libcgroup default group does not have realtime access, so processes expecting to be able to run in a realtime scheduling class will be disappointed. 

As it happens, ~~Ubuntu 10.10~~ the upcoming Ubuntu 11.04 release installs and configures libcgroup in just this mode. That causes trouble for Ubuntu users running JACK-based audio configurations; audio dropouts are not the "perfect 10" experience they had been hoping for. In response, there has been quite a bit of complaining on the JACK list, most of which has been aimed at the kernel. But it is not, in fact, a kernel problem; the kernel is behaving exactly as intended - a fact which has not made JACK developers feel any better. 

As libcgroup developer Dhaval Giani [pointed out](<https://lwn.net/Articles/420410/>), there are a few ways to solve this problem. The easiest is to simply turn off the default group feature with a one-line configuration change; only slightly less easy is enabling realtime access for that default group. The best solution, according to Dhaval, is to create a separate control group for JACK which would provide realtime access to just the processes which need it. That solution is slightly trickier than he had imagined, mostly because JACK clients are not necessarily started by JACK itself, so they won't be in the special JACK group by default. There are ways of getting around this difficulty, but they may require Linux-specific application changes. 

The JACK developers were not greatly mollified by this information; in their view, audio developers have been getting the short end of the stick from the kernel community for years, and this change is just more of the same. They would, it seems, rather stick with the solution they have, which has been working for a few years now. As [Paul Davis put it](<https://lwn.net/Articles/420411/>): 

But I hope you can perhaps understand how incredibly irritating it is that *just* as almost [all] mainstream distros now finally come with the mechanism to grant ordinary users SCHED_FIFO and memlock without too much hassle (its taken more than 8 years to get there), RT_GROUP_SCHED appears without any apparent appreciation for the impact on what is probably the most widely used RT-scheduled application "ecosystem" on Linux. 

Many of the other thoughts expressed on the list were rather less polite. The audio development community, it seems, feels that it is not being treated with the respect that it deserves. 

It is true that the audio folks have had a bit of a hard time of it. They have made a few attempts to engage with the kernel community which have been less than successful; since then, they have mostly just had to accept what came their way. And what has come their way has not always been what they felt they needed. As [expressed by Alex Stone](<https://lwn.net/Articles/420414/>), the audio community clearly feels that the kernel developers should be paying more attention: 

So no-one thought, while building this exciting new feature, to do a quick test, or at least have a think about, of the significance of the impact on jack/RT, given the nature of the feature as a scheduler, and what many users think is JACK and jack based apps importance in the linux community? 

Sort of confirms the indifference to jack/RT as a significant component in the linux audio/midi/video world, doesn't it? 

One other sentence in Alex's message deserves special attention, though: ""If we don't yell, we don't get considered?"" The answer to that question is "yes." The kernel serves a huge community of users, many of whom are represented within the kernel development community. It is entirely unsurprising that groups which don't "yell" tend to find that their needs are not always met. Any group which declines to participate, feeling instead that it's so important that kernel developers should come to them, is bound to be disappointed with future kernels. We all have to yell when our toes are stepped on; the sooner we yell the better the results will be. 

That said, no amount of yelling at the kernel will help when the problem is elsewhere. Ubuntu has created a configuration in which allowing unprivileged access to realtime scheduling requires a bit more administrative work than it did before. Fedora, which also installs libcgroup, has, perhaps accidentally, avoided this problem by not enabling the "default group" option. So one might say that Ubuntu would be an appropriate target for any yelling on this topic. But increased use of control groups is clearly on the horizon for a number of distributions; systemd depends on them heavily. So the realtime audio community will need to work with control groups, like it or not. The good news is that control groups provide the needed features, and they do it in a way which is more secure and which allows more control over policy. 

The JACK community seems to have figured this out; there have already been some patches posted to give JACK an understanding of control groups. It would also appear that the libcgroup developers are [working on the problem](<https://lwn.net/Articles/420415/>) in the hope of producing a solution which doesn't require application changes. Then, hopefully, Linux audio developers will have a solution which they can expect to rely on for many years (though they will want to keep an eye on the progress of the deadline scheduling patches). Certainly this kind of solution is something they have been wanting for a long time. 

(Thanks to David Nielson for the heads-up).
