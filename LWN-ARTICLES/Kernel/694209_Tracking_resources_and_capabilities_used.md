---
title: Tracking resources and capabilities used
url: https://lwn.net/Articles/694209/
date: "July 13, 2016"
category: Capabilities; Control groups
author: "By Jake Edge July 13, 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
July 13, 2016

There are various types of limits and privileges that administrators can apply to processes or control groups (cgroups) in Linux, but it is sometimes difficult to determine what those values should be—except by trial and error. A [patch set](<https://lwn.net/Articles/691112/>) from Topi Miettinen targets making that easier by tracking resource and capability usage by processes in order to give users and administrators a starting point to use when setting those values. The idea is that the processes can be run under a normal load and the high-water values (as well as the capabilities used) will be recorded to provide a guide for future, more-restrictive deployments. 

The 18-patch series is broken up into three groups: capabilities used (one patch), cgroup limits (three patches), and resource limits (14 patches). Capabilities used are reported in `/proc/PID/status`, while cgroup maximums are presented in files in the cgroup filesystem. Resource limits (i.e. [rlimits](<http://man7.org/linux/man-pages/man2/prlimit.2.html>)), on the other hand, are reported in the `/proc/PID/limits` file. Those may change since there are programs that parse the files in `/proc`, so adding more information could potentially alter the user-space interface for the kernel. 

As Miettinen says in the cover letter for the patches, much of the information can already be gleaned from various `/proc` files and using tools like `ps`, but those methods only give a value at one point in time. In order to be sure that transient spikes are also recorded, so they can be taken into account, the kernel needs to be involved; thus these patches. 

But Konstantin Khlebnikov [objected](<https://lwn.net/Articles/694250/>) to the overall goal: 

All limitations are context dependent and that context changes rapidly. You'll never dump enough information for predicting future errors or investigating [the reason] of errors in past. 

He also suggested that tracepoints could be used (perhaps in conjunction with SystemTap or other kernel tracing infrastructure), rather than adding high-water recording to the kernel. 

But both Miettinen and Austin S. Hemmelgarn disagreed with that analysis. Miettinen [noted](<https://lwn.net/Articles/694252/>) that there are always risks when setting limits, but that the patches are just meant to help provide some guidance. Hemmelgarn [essentially agreed](<https://lwn.net/Articles/694253/>): 

It's still better than what we have now, and there is one particular use for the cgroup stuff that I find intriguing, you can create a cgroup, populate it, set no limits, and then run a simulated workload against it and see how it reacts. This in general will probably provide a better starting point for what to actually set the limits to than just making an arbitrary guess. 

Rlimits could be handled similarly, he said. Beyond that, though, there are different types of failure modes for processes that cannot get the resources they need (e.g. can't start a thread or process), which may not manifest as application errors or crashes. In addition, getting the information about the maximum usage from user space will be difficult or impossible, he said. In a follow-up [post](<https://lwn.net/Articles/694256/>), he also noted that tracing can't supply any better answers for the upper bound of these values than internal kernel tracking can: ""You can't get a perfectly reliable upper bound for any type of resource usage with just black box observations, period."" 

There were also comments on many of the individual patches. The [capabilities-tracking patch](<https://lwn.net/Articles/694257/>) simply adds a `cap_used` bit array to `struct task_struct` and sets the bit corresponding to a capability whenever that capability is checked (and passes the check). But as Andy Lutomirski [pointed out](<https://lwn.net/Articles/694270/>), simply tracking the capabilities used by a process won't work well in the presence of [ambient capabilities](<https://lwn.net/Articles/632520/>). If a process runs a program with ambient capabilities, which uses some capabilities beyond what the main process uses, those will be missed in the set of capabilities collected. He suggested tracking capabilities used for an entire process tree or cgroup. 

The cgroup patches track values for three specific controllers: the maximum PIDs used in a PID cgroup, maximum memory used in a memory cgroup, and the devices accessed in a device cgroup. The PID cgroup [patch](<https://lwn.net/Articles/694275/>) uses an atomic variable to track the highest number of PIDs that have been active in the cgroup at any point. It makes that number available in the `pids.current_max` file. Cgroup maintainer Tejun Heo [didn't like the name](<https://lwn.net/Articles/694276/>) (he suggested a `high_watermark` field in the `pids.stats` file) and was concerned that some of the atomic variable handling that could lead to races. 

The [patch](<https://lwn.net/Articles/694277/>) for the memory cgroup simply presents the existing `watermark` value in the `memory.current_max` file. But, as Johannes Weiner [noted](<https://lwn.net/Articles/694278/>), that generally won't provide much useful information. The page cache is counted in that watermark and is not reduced in size unless there is memory pressure, ""so in all but very few cases the high watermark you are introducing will be pegged to the configured limit"". 

The [last](<https://lwn.net/Articles/694286/>) of the cgroup patches keeps a list of devices that are accessed in a device cgroup. That list, which contains the device type (character or block), major and minor numbers, and access type (read, write, or mknod), can be read from the `devices.accessed` file. 

The rlimit patches drew fewer comments in general (or, perhaps, the comments were outweighed by the sheer number of patches). There was some general confusion because Miettinen did not send a copy of the cover letter (or the first rlimit [patch](<https://lwn.net/Articles/694288/>) that added some infrastructure used by the rest) to everyone who got copies of the individual patches. In addition, the function name used to update the current maximum value, `bump_rlimit()`, was confusing to some, since it seems to imply that the actual rlimit is being increased (bumped). 

There are individual patches to record (and sometimes report) the maximum use of different resources that are tied to rlimits. That includes the number of open files (`RLIMIT_NOFILE`), CPU usage (`RLIMIT_CPU`), file sizes created (`RLIMIT_FSIZE`), number of processes (`RLIMIT_NPROC`), and so on. There were some complaints about race conditions and using read-copy-update (RCU) incorrectly, along with some suggestions for better comments to make the intent of the code clearer. Aside from the [final patch](<https://lwn.net/Articles/694304/>) in the series, which Kees Cook [pointed out](<https://lwn.net/Articles/694305/>) was unneeded, the series as a whole got a fairly warm response. 

There is clearly some work to be done, but maximum resource usage tracking seems like a feature that might make its way into the kernel in, say, 4.9 or 4.10 unless some major opposition appears. It will provide users with a way to gauge what their processes are doing so that limits and privileges can be tightened down appropriately. It certainly won't provide all the answers, but may give the starting point that Miettinen is seeking.
