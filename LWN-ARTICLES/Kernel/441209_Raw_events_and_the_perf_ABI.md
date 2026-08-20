---
title: Raw events and the perf ABI
url: https://lwn.net/Articles/441209/
date: "May 3, 2011"
category: Performance monitoring
author: "By Jonathan Corbet May 3, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 3, 2011

The perf events subsystem often looks like it's on the path to take over the kernel; there is a great deal of development activity there, and it has become a sort of generalized event reporting mechanism. But the original purpose of perf events was to provide access to the performance monitoring counters made available by the hardware, and it is still used to that end. The merging of perf was a bit of a hard pill for users of alternative performance monitoring tools to swallow, but they have mostly done so. The recent discussion on "offcore" events shows that there are still some things to argue about in this area, even if everybody seems likely to get what they want in the end. 

The performance monitoring unit (PMU) is normally associated with the CPU; each processor has its own PMU for monitoring its own specific events. Some newer processors (such as Intel's Nehalem series) also provide a PMU which is not tied to any CPU; in the Nehalem case it's part of the "uncore" which handles memory access at the package level. The off-core PMU has a viewpoint which allows it to provide a better picture of the overall memory behavior of the system, so there is interest in gaining access to events from that PMU. Current kernels, though, do not provide access to these offcore events. 

For a while, the 2.6.39-rc kernel _did_ provide access to these events, following the merging of [a patch by Andi Kleen](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=e994d7d23a0bae34cd28834e85522ed4e782faf7>) in March. One piece that was missing, though, was a patch to the user-space `perf` tool to provide access to this functionality. There was an attempt to merge that piece toward the end of April, but it did not yield the desired results; rather than merge the additional change, perf maintainer Ingo Molnar [removed the ability to access offcore events](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=b52c55c6a25e4515b5e075a989ff346fc251ed09>) entirely. 

Needless to say, that action has led to some unhappiness in the perf user community; there are developers who had already been making use of those events. Normally, breaking things in this way would be considered a regression, and the patch would be backed out again. But, since this functionality never appeared in a released kernel, it cannot really be called a regression. That, of course, is part of the point of removing the feature now. 

Ingo's complaint is straightforward: the interface to these events was too low-level and too difficult to use. The rejected perf patch had an example command which looked like: 

```
perf stat -e r1b7:20ff -a sleep 1
```

Non-expert readers may, indeed, be forgiven for not immediately understanding that this command would monitor access to remote DRAM - memory which is hosted on a different socket. Ingo asserted that the feature should be more easily used, perhaps with a command like (from the patch removing the feature): 

```
perf record -e dram-remote ./myapp
```

He also said: 

But this kind of usability is absolutely unacceptable - users should not be expected to type in magic, CPU and model specific incantations to get access to useful hardware functionality. 

The proper solution is to expose useful offcore functionality via generalized events - that way users do not have to care which specific CPU model they are using, they can use the conceptual event and not some model specific quirky hexa number. 

The key is the call for "generalized events" which are mapped, within the kernel, onto whatever counters the currently-running hardware uses to obtain that information. Users need not worry about the exact type of processor they are running on, and they need not dig out the data sheet to figure out what numbers will yield the results they want. 

Criticism of this move has taken a few forms. Generalized events, it is said, are a fine thing to have, but they can never reflect all of the weird, hardware-specific counters that each processor may provide. These events should also be managed in user space where there is more flexibility and no need to bloat the kernel. There were some complaints about how some of the existing generalized events have not always been implemented correctly on all architectures. And, they say, there will always be people who want to know what's in a specific hardware counter without having the kernel trying to generalize it away from them. As Vince Weaver [put it](<https://lwn.net/Articles/441218/>): 

Blocking access to raw events is the wrong idea. If anything, the whole "generic events" thing in the kernel should be ditched. Wrong events are used at times (see AMD branch events a few releases back, now Nehalem cache events). This all belongs in userspace, as was pointed out at the start. The kernel has no business telling users which perf events are interesting, or limiting them! 

Ingo's [response](<https://lwn.net/Articles/441220/>) is that the knowledge and techniques around performance monitoring should be concentrated in one place: 

Well, the thing is, i think users are helped most if we add useful, highlevel PMU features added and not just an opaque raw event pass-through engine. The problem with lowlevel raw ABIs is that the tool space fragments into a zillion small hacks and there's no good concentration of know-how. I'd like the art of performance measurement to be generalized out, as well as it can be. 

Vince, meanwhile, [went on](<https://lwn.net/Articles/441226/>) to claim that perf was a reinvention of the wheel which has ignored a lot of the experience built into its predecessors. There are, it seems, still some scars from that series of events. Thomas Gleixner [disagreed with](<https://lwn.net/Articles/441227/>) the claim that perf is an exercise in wheel reinvention, but he did say that he thought the raw events should be made available: 

The problem at hand which ignited this flame war is definitely borderline and I don't agree with Ingo that it should not made be available right now in the raw form. That's an hardware enablement feature which can be useful even if tools/perf has not support for it and we have no generalized event for it. That's two different stories. perf has always allowed to use raw events and I don't see a reason why we should not do that in this case if it enables a subset of the perf userbase to make use of it. 

It turns out that [Ingo is fine with raw events too](<https://lwn.net/Articles/441220/>). His stated concern is that access to raw events should not be the primary means by which most users gain access to those performance counters. So he is blocking the availability of those events for now for two reasons. One of those is that he wants the generalized mode of access to be available first so that users will see it as the normal way to access offcore events. If there is never any need to figure out hexadecimal incantations, many user-space developers will never bother; as a result, their commands and code should eventually work on other processors as well. 

The other reason for blocking raw events now is that, as the interface to these events is thought through, the ABI by which they are exposed to user space may need to change. Releasing the initial ABI in a stable kernel seems almost certain to cement it in place, given that people were already using it. By deferring these events for one cycle (somebody will certainly come up with a way to export them in 2.6.40), he hopes to avoid being stuck with a second-rate interface which has to be supported forever. 

There can be no doubt that managing this feature in this way makes life harder for some developers. The kernel process can be obnoxious to deal with at times. But the hope is that doing things this way will lead to a kernel that everybody is happier with five years from now. If things work out that way, most of us can deal with a bit of frustration and a one-cycle delay now.
