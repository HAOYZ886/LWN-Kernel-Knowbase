---
title: The case of the stalled CPU controller
url: https://lwn.net/Articles/697366/
date: "August 17, 2016"
category: "Control groups-Thread-level control"
author: "By Jonathan Corbet August 17, 2016"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 17, 2016

Long-running technical disagreements are certainly well-known in the kernel community. Usually they are eventually resolved and the developers involved move on to new problems. Occasionally, though, stalled consensus can lead to a break between the kernel and its user community. The ongoing dispute over the CPU controller in the new control-group hierarchy is beginning to look like one of those unpleasant cases. 

Control groups (cgroups) have been supported by the kernel for nearly ten years; they provide a mechanism by which processes can be grouped in a hierarchical manner and made subject to various resource controllers. It did not take long after the introduction of control groups before users and developers began to realize that there were some fundamental problems with its design; the discussion about fixing those problems [got started in earnest](<https://lwn.net/Articles/484251/>) in early 2012. Those discussions led to the description of a second version of the cgroup API and the beginning of work to move to that API. At this point, the [version-2 API](<https://lwn.net/Articles/671722/>) transition is mostly complete, with one glaring exception: the CPU controller. 

The CPU controller, as one might expect, controls access to the CPU; it allows different groups of processes to be allocated specific amounts of CPU time and keeps those groups from interfering with each other. The low-level CPU-controller code is able to support the new API without trouble, but the scheduler developers have resisted the merging of that API itself; at this point, the CPU controller is the most significant controller without version-2 support. In an attempt to push things forward (and to say what will happen if things do not move forward), cgroup maintainer Tejun Heo posted [a detailed summary](<https://lwn.net/Articles/697369/>) of the situation as he sees it. That document is well worth reading for those who are interested in the topic. 

#### Objections to the CPU controller

In short, the scheduler developers object to the new API for two reasons, both stemming from a perceived mismatch between the API and how they feel CPU control should be done. Both relate to fundamental design decisions in the version-2 cgroup API. 

In the original cgroup implementation, each thread (of which a process may contain many) can be placed in a separate control group. The version-2 API, instead, requires that all of a process's threads be in the same group. For some controllers, such as the memory-usage controller, putting different threads into different groups makes little sense; all those threads are sharing the same memory, after all, so it is hard to say what it would mean to try to apply different policies to different threads. There _are_ reasonable use cases for applying different CPU-usage policies to different threads, but the unified hierarchy, which is a fundamental design aspect of the version-2 API, requires all controllers to see the same cgroup arrangement. So all threads must be in the same cgroup from the CPU controller's point of view. 

This requirement apparently seems fundamentally wrong to the scheduler developers; nothing in the scheduler itself recognizes the abstraction of a "process" at all. At that level, everything is a thread; applying a coarser policy at the cgroup level takes away an important degree of flexibility for (from their point of view) no gain. 

There are users who want to be able to apply different policies to different threads; managing a thread pool is [one commonly cited use case](<https://lwn.net/Articles/697376/>). But Heo stands by the design decisions; he also feels that the same interface should not be used at both the administrator level and within an individual process. He has proposed a mechanism called [resource groups](<https://lwn.net/Articles/679940/>) for the intra-process case, but that proposal has not made a lot of headway thus far. 

There is another version-2 design decision that does not sit well with the scheduler developers. In the new API, a control group may contain other control groups, or it may contain processes, but not both; processes can only appear as leaves in the control-group hierarchy. Again, this decision was made to facilitate support for controllers other than the CPU controller. If subgroups and processes appear in the same cgroup, then the two types of object must compete for the same resource. In the CPU case, that competition is easily managed; when a cgroup is "scheduled," the scheduler recurses into the group and chooses one of the entities found therein to run. For many other controllers, though, it is not possible to treat processes and subgroups in the same manner. 

The primary objection here seems to be that this restriction stomps on some of the elegance in the CPU-scheduler design; scheduling decisions are applied to "scheduling entities" that can be either processes or groups, and the scheduler itself need not care which. The version-2 API makes some control policies difficult or impossible to achieve but, Heo asserts, that may not matter much: 

However, it isn't clear what the practical usefulness of a layout with direct competition between tasks and cgroups would be, considering that number and behavior of tasks are controlled by each application, and cgroups primarily deal with system level resource distribution; changes in the number of active threads would directly impact resource distribution. Real world use cases of such layouts could not be established during the discussions. 

In summary, Heo says, there are solid reasons for the decisions that were made in the version-2 API. It handles most use cases as-is, and the addition of features like resource groups can fill in the gaps that remain. If there is anybody who still cannot work with the version-2 API, version-1 will continue to be maintained for as long as it has users. The transition is nearly done: the low-level support is there; all that is left to be merged is the API-level code to allow the CPU controller to operate in the unified version-2 hierarchy. But that code has been blocked with, seemingly, no way forward. 

#### What happens now

Heo clearly hopes that, by reopening the discussion, he can maybe bring it to a conclusion and clear the way for the remaining patches to be merged. There is little evidence of that happening from the discussion so far, though. In the absence of a solution there, he is planning to do a couple of other things to make this functionality available to users. 

One of those is to maintain the necessary patches going forward so that anybody who wants the CPU controller with the version-2 API can easily add it to their kernel. While it is unstated, it seems fairly clear that he is hoping that distributors will apply these patches to make the functionality available to their users. That approach has been used to resolve such logjams in the past; if a patch is widely applied by distributors and widely used, there comes a point where it clearly makes no sense to keep it out of the mainline. That was part of the reasoning that brought the Android patches into the mainline, among others. 

The other half of that picture is to ensure that the most widely distributed user of control groups — systemd — is able to use the version-2 API. To that end, he has posted [a pull request](<https://github.com/systemd/systemd/pull/3905>) to add this functionality to systemd, saying: ""This commit implements systemd CPU controller support on the unified hierarchy so that users who choose to deploy CPU controller cgroup v2 support can easily take advantage of it."" That code was merged into the systemd mainline on August 14\. 

That action has led to a bit of [disagreement](<https://lwn.net/Articles/697379/>) in the systemd community, given that systemd normally wants to see features merged upstream before adding code to make use of them — though, it must be said, the bulk of that disagreement seems to come from a single vociferous developer. Lennart Poettering [defended](<https://lwn.net/Articles/697380/>) the action, saying that the systemd developers want to get the capability into users' hands, and that he hopes to get the kernel patches added to Fedora's kernel as well. Greg Kroah-Hartman [added](<https://lwn.net/Articles/697381/>) that this is not the first time that support for unmerged features has been added, and that it is often for good reasons: 

Sometimes you have to add code to projects in order to be able to properly test the kernel code. And to make it easier for people to upgrade their kernels in the future and have things work properly on their existing, older, system tools. This happens all the time, I don't know why you are suddenly surprised about this 

That is where things stand as of this writing. Predictions can be dangerous, especially when they involve the future, but, in this case, it seems likely that the kernel patches will indeed find their way into a number of distributor kernels. They make the version-2 API more widely useful, and, since most distributors are using systemd at this point, they have an important consumer lined up and ready to use it. Pressure from the user community is a blunt tool to use when patches are stalled but, in this case, it might just work.
