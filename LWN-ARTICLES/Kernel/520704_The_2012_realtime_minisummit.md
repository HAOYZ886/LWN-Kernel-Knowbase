---
title: The 2012 realtime minisummit
url: https://lwn.net/Articles/520704/
date: "October 24, 2012"
category: "Interrupts-Software; Realtime"
author: "By Jake Edge October 24, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
October 24, 2012

As is generally the case when realtime Linux developers get together, the discussion soon turns to how (and when) to get the remaining pieces of the realtime patch set into the mainline. That was definitely the case at the 2012 realtime minisummit, which was held October 18 in conjunction with the [14th Real Time Linux Workshop](<https://www.osadl.org/RTLWS-2012.rtlws-2012.0.html>) (RTLWS) in Chapel Hill, North Carolina. Some other topics were addressed as well, of course, and a lively discussion, which Thomas Gleixner characterized as "twelve people siting around a table not agreeing on anything", ensued. Gleixner's joke was just that, as there was actually a great deal of agreement around that table. 

I unfortunately missed the first hour or so of the minisummit, so I am using Darren Hart's notes, Gleixner's recap for the entire workshop on October 19, and some conversations with attendees as the basis for the report on that part of the meeting. 

[ ![\[RTLWS group photo\]](https://static.lwn.net/images/2012/rtlws-group-sm.jpg) ](<https://lwn.net/Articles/521023/>)

#### Development process

The first topic was on using Bugzilla to track bugs in the realtime patches. Hart and Clark Williams have agreed to shepherd a Bugzilla to help ensure the bugs have useful information and provide the needed pieces for the developers to track the problems down. Bugs can now be reported to the [kernel Bugzilla](<http://bugzilla.kernel.org/>) using `PREEMPT_RT` for the "Tree" field. Doing so will send an email to developers who have registered their interest with Hart. 

Gleixner has "mixed feelings" about it because it involves "web browsers, mouse clicks and other things developers hate". Previously, the normal way to report a bug was via the realtime or kernel mailing lists, but Bugzilla does provide a way to attach large files (e.g. log files) to bugs, which may prove helpful. The realtime hackers will know better in a year how well Bugzilla is working out and will report on it then, he said. 

There was general agreement that the development process for realtime is working well. Currently, Gleixner is maintaining a patch set based on 3.6, which will be turned over to Steven Rostedt when it stabilizes. Rostedt then follows the mainline stable releases and is, in effect, the stable "team" for realtime. Those stable kernels are the ones that users and distributions generally base their efforts on. In the future, Gleixner has plans to update his 3.6-rt tree with incremental patches that have already been merged into other stable realtime kernels (3.0, 3.2, 3.4) to keep it closer to the mainline 3.6 stable release. 

There was some discussion of the [long-term support initiative](<https://lwn.net/Articles/484337/>) (LTSI) kernels and what relationship those kernels have with the realtime stable kernels. The answer is: not much. LTSI plans to have realtime versions of its kernels, but when Hart suggested aligning the realtime kernel versions with those of LTSI, it was not met with much agreement. Gleixner said that the LTSI kernels would likely be supported for years, "probably decades", which is well beyond the scope of what the realtime developers are interested in doing. 

#### 3.6 softirq changes

One of the topics that came up frequently as part of both the workshop/minisummit and the extensive hallway/micro-brewery track was Gleixner's [softirq processing changes](<https://lwn.net/Articles/520076/>) released in 3.6-rt1. The locks for the ten different softirq types have been separated so that the softirqs raised in the context of a thread can be handled in that thread--without having to handle unrelated softirqs. This solves a number of problems with softirq handling (victimizing unrelated threads to process softirqs, configuring separate softirq thread priorities to get the desired behavior, etc.), but is a big change from the existing mainline implementation--as well as from previous realtime patch sets. 

In the minisummit, Gleixner emphasized that more testing of the patches is needed. Networking, which is the most extensive user of softirqs in the kernel, needs more testing in particular. But the larger issue is the possibility of eventually eliminating softirqs in the kernel completely. To that end, each of the specific kernel softirq-using subsystems was discussed, with an eye toward eliminating the softirq dependency for both realtime and mainline. 

The use of softirqs in the network subsystem is "massive" and even the network developers are not quite sure why it all works, according to Gleixner. But, softirqs seem to work fine for Linux networking, though the definition of "working" is not necessarily realtime friendly. If the kernel can pass the network throughput tests and fill the links on high-speed test hardware, then it is considered to be working. Any alternate solution will have to meet or exceed the current performance, which may be difficult. 

The block subsystem's use of softirqs is mostly legacy code. Something like 90% of the deferred work has been shifted to workqueues over the years. Eliminating the rest won't be too difficult, Gleixner said. 

The story with tasklets is similar. They should be "easy to get rid of", he said, it will just be a lot of work. Tasklets are typically used by legacy drivers and are not on a performance-critical path. Tasklet handling could be moved to its own thread, Rostedt suggested, but Gleixner thought it would be better to eliminate them entirely. 

The timer softirq, which is used for the timer wheel (described and diagrammed in this [LWN article](<https://lwn.net/Articles/152436/>)), is more problematic. The timer wheel is mostly used for timeouts in the network stack and elsewhere, so it is pretty low priority. It can't run with interrupts disabled in either the mainline or in the realtime kernel, but it has to run somewhere, so pushing it off to `ksoftirqd` is a possibility. 

The high-resolution timers softirq is mostly problematic because of POSIX timers and their signal-delivery semantics. Determining which thread should be the "victim" to deliver the signal to can be a lengthy process, so it is not done in the softirq handler in the realtime patches as it is in mainline. One solution that may be acceptable to mainline developers is to set a flag in the thread which requested the timer, and allow it to do all of the messy victim-finding and signal delivery. That would mean that the thread which requests a POSIX timer pays the price for its semantics. 

Williams asked if users were not being advised to avoid signal-based timers. Gleixner said that he tells users to "use pthreads". But, "customers aren't always reasonable", Frank Rowand observed. He pointed out that some he knows of are using floating point in the kernel, and now that they have hardware floating point want to add that context to what is saved during context switches. Paul McKenney noted that many processors have lots of floating point registers which can add "multiple hundreds of ~~milliseconds~~ microseconds" to save or restore. Similar problems exist for the auto-vectorization code that is being added to GCC, which will result in many more registers needing to be saved. 

Back to the softirqs, McKenney said that the read-copy-update (RCU) work had largely [moved to threads in 3.6](<https://lwn.net/Articles/518953/>), but that not all of the processing moved out of the softirq. He had tried to completely move out of softirq in a patch a ways back, but Linus Torvalds "kicked it out immediately". He has some ideas of ways to address those complaints, though, so eliminating the RCU softirq should be possible. 

Finally, the scheduler softirq does "nothing useful that I can see", Gleixner said. It mostly consists of heuristics to do load balancing, and Peter Zijlstra may be amenable to moving it elsewhere. Mike Galbraith pointed out that the [NUMA scheduling](<https://lwn.net/Articles/490114/#numa>) work will make the problem worse, as will power management. ARM's [big.LITTLE scheduling](<https://lwn.net/Articles/501501/>) could also complicate things, Rowand said. 

There is a great deal of interest in getting those changes into the 3.2 and 3.4 realtime kernels. Later in the meeting, Rostedt said that he would create an unstable branch of those kernels to facilitate that. The modifications are "pretty local", Gleixner said, so it should be fairly straightforward to backport the changes. In addition, it is unlikely that backports of other fixes into the mainline stable kernels (which are picked up by the realtime stable kernels) will touch the changed areas, so the ongoing maintenance should not be a big burden. 

#### Upstreaming

Gleixner said that he is "swamped" by a variety of tasks, including stabilizing the realtime tree, the softirq split, and a "huge backlog" of work that needs to be done for the CPU hotplug rework. Part of the latter was merged for 3.7, but there is lots more to do. Rusty Russell has offered to help once Gleixner gets the infrastructure in place, so he needs to "get that out the door". Beyond that, he also spends a lot of time tracking down bugs found by the Open Source Automation Development Lab (OSADL) testing and from Red Hat bug reports. 

He needs some help from the other realtime kernel developers in order to move more of the patch set into the mainline. Those in the room seemed very willing to help. The first step is to go through all of the realtime patches and work on any that are "halfway reasonable to get upstream" first. 

One of the top priorities for upstreaming is not a kernel change, but is a change needed in the GNU C library (glibc). Gleixner noted that the development process for glibc has gotten a "lot better" recently and that the new maintainers are doing a "great job". That means that a longstanding [problem with condvars and priority inheritance](<http://sourceware.org/bugzilla/show_bug.cgi?id=11588>) may finally be able to be addressed. 

When priority inheritance was added to the kernel, Ulrich Drepper wrote the user-space portion for glibc. He had a solution for the problem of condvars not being able to specify that they want to use a priority-inheriting mutex, but that solution was one that Gleixner and Ingo Molnar didn't like, so nothing was added to glibc. 

Three years ago, Hart presented a solution at the RTLWS in Dresden, but he was unable to get it into glibc. It is a real problem for users according to Gleixner and Williams, so Hart's solution (or something derived from it) should be merged into glibc. Hart said he would put that at the top of his list. 

#### SLUB

Another area that should be fairly easy to get upstream are changes to the SLUB allocator to make it work with the realtime code. SLUB developer Christoph Lameter has done some work to make the core allocator lockless and for it not to disable interrupts or preemption. Lameter's work was mostly to support enterprise users on large NUMA systems, but it should also help make SLUB work better with realtime. 

If SLUB can be made to work relatively easily, Gleixner would be quite willing to drop support for SLAB. The SLOB allocator is targeted at smaller, embedded systems, including those without an MMU, so it is not an interesting target. Besides which, SLOB's "performance is terrible", Rostedt said. During the minisummit, Williams was able to build and boot SLUB on a realtime system, which "didn't explode right away", Gleixner reported in the recap. That, coupled with SLUB's better NUMA performance, may make it a much better target anyway, he said. 

Switching to SLUB might also get rid of a whole pile of "intrusive changes" in the memory allocator code. The realtime memory management changes will be some of the hardest to sell to the upstream developers, so any reduction in the size of those patches will be welcome. 

There are a number of places where drivers call `local_irq_save()` and `local_irq_enable()` that have been changed in the realtime tree to call `*_nort()` variants. There are about 25 files that use those variants, mostly drivers designed for uniprocessor machines that have never been fixed for multiprocessor systems. No one really cares about those drivers any more, Gleixner said, so the `_nort` changes can either go into mainline or be trivially maintained out of it. 

#### Spinlocks

Bit spinlocks (i.e. single bits used as spinlocks) need to be changed to support realtime, and that can probably be sold because it would add lockdep coverage. Right now, bit spinlocks are not checked by lockdep, which is a debugging issue. In converting bit spinlocks to regular spinlocks, Gleixner said he found 3-4 locking bugs in the mainline, so it would be beneficial to have a way to check them. 

The problem is that bit spinlocks are typically using flag bits in size-constrained structures (e.g. `struct page`). But, for debugging, it will be acceptable to grow those structures when lockdep is enabled. For realtime, there is a need to just "live with the fact that we are growing some structures", Gleixner said. There aren't that many bit spinlocks; two others that he mentioned were the buffer head lock and the journal head lock. 

Hart brought up the sleeping spinlock conversion, but Gleixner said that part is the least of his worries. Most of the annotations needed have already been merged, as have the header file changes. The patches are "really unintrusive now", though it is still a big change. 

The CPU hotplug rework should eliminate most of the changes required for realtime once it gets merged. The [migrate enable and disable patches](<https://lwn.net/Articles/452884/>) are self-contained. The high-resolution timers changes and softirq changes can be fairly self-contained as well. Overall, getting the realtime patches upstream is "not that far away", Gleixner said, though some thought is needed on good arguments to get around the "defensive list" of some mainline developers. 

To try to ensure they hadn't skipped over anything, Williams put up a January email from Gleixner with a "to do" list for the realtime patches. There are some `printk()`-related issues that were on the list. Gleixner said those still linger, and it will be "messy" to deal with them. 

Zijlstra was at one time opposed to the explicit migrate enable/disable calls, but that may be not be true anymore, Gleixner said. The problem may be that there will be a question of who uses the code when trying to get the infrastructure merged. It is a "hen-egg problem", but there needs to be a way to ensure that processes do not move between CPUs, particularly when handling per-CPU variables. 

In the mainline, spinlocks disable preemption (which disables migration), but that's not true in realtime. The current mainline behavior is somewhat "magic", and realtime adds an explicit way to disable migration if that's truly what's needed. As Paul Gortmaker put it, "making it explicit is an argument for it in it's own right". Gleixner said he would talk to Zijlstra about a use case and get the code into shape for mainline. 

Gortmaker asked if there were any softirq uses that could be completely eliminated. McKenney believes he can do so for the RCU softirq, but he does have the reservation that he has never successfully done so in the past. High-resolution timers and the timer wheel can both move out of softirqs, Gleixner said, though the former may be tricky. The block layer softirq work can be moved to workqueues, but the network stack is the big issue. 

One possible solution for the networking softirqs is something Rostedt calls "ENAPI" (even newer API, after "NAPI", the new API). When using threaded interrupt handlers, the polling that is currently done in the softirq handler could be done directly in the interrupt handler thread. If that works, and shows a performance benefit, Gleixner said, the network driver writers will do much of the work on the conversion. 

Wait queues are another problem area. While most are "pretty straightforward", there are some where the wait queue has a callback that is called on wakeup for every woken task. Those callbacks could do most anything, including sleep, which prevents those wait queues from being converted to use raw locks. Lots of places can be replaced, but not for "NFS and other places with massive callbacks", Gleixner said. 

There are a number of pieces that should be able to go into mainline largely uncontended. Code to shorten the time that locks are held and to reduce the interrupt and preempt disabled regions is probably non-controversial. The `_nort` annotations may also fall into that category as they don't hurt things in mainline. 

#### CPU isolation

The final item on the day's agenda is a feature that is not part of the realtime patches, but is of interest to many of the same users: CPU isolation. That feature, which is known by other names such as "adaptive NOHZ", would allow users to dedicate one or more cores to user-space processing by removing all kernel processing from those cores. Currently, nearly all processing can be moved to other cores using CPU affinity, but there is still kernel housekeeping (notably CPU time accounting and RCU) that will run on those CPUs. 

Frédéric Weisbecker has been working on CPU isolation, and he attended the minisummit at least partly to give an update on the status of the feature. Accounting for CPU time without the presence of the timer tick is one of the areas that needs work. Users still want to see things like load averages reflect the time being spent in user-space processing on an isolated CPU, but that information is normally updated in the timer tick interrupt. 

In order to isolate the CPU, though, the timer tick needs to be turned off. In the recap, Gleixner noted that the high-performance computer users of the feature aren't so concerned about the time spent in the timer tick (which is minimal), but the cache effects from running the code. Knocking data and instructions out of the cache can result in a 3% performance hit, which is significant for those workloads. 

To account for CPU time usage without the tick, adaptive NOHZ will use the same hooks that RCU uses to calculate the CPU usage. While the CPU is isolated, the CPU time will just be calculated, but won't be updated until the user-space process enters the kernel (e.g. via a system call). The tick might be restarted when system calls are made, which will eventually occur so that the CPU-bound process can report its results or get new data. Restarting the tick would allow the CPU accounting and RCU housekeeping to be done. Weisbecker felt that it should only be restarted if it was needed for RCU; even that might possibly be offloaded to a separate CPU. 

That led to a discussion of what the restrictions there are for using CPU isolation. There was talk of trying to determine which system calls will actually require restarting the tick, but that was deemed too kernel-version specific to be useful. The guidelines will be that anything other than one thread that makes no system calls on the CPU may result in less than 100% of the CPU available. Gleixner suggested adding a tracepoint that would indicate when the CPU exited isolated mode and why. McKenney suggested a warning like "this code needs a more deterministic universe"--to some chuckles around the table. Weisbecker and Rostedt plan to work on CPU isolation in the near term, with an eye toward getting it upstream soon. 

And that is pretty much where the realtime minisummit ended. While there is plenty of work still to do, it is clear that there is increasing interest in "finishing" the task of getting the realtime changes merged. Gleixner confessed to being tired of maintaining it in the recap session, and that feeling is probably shared by others. Given that the mainline has benefited from the realtime changes already merged, it seems likely that will continue as more goes upstream. The trick will be in convincing the other kernel hackers of that. 

[ I would like to thank OSADL and RTLWS for supporting my travel to the minisummit and workshop. ]
