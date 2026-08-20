---
title: Notes from the LPC scheduler microconference
url: https://lwn.net/Articles/734039/
date: "September 18, 2017"
category: "Development tools-rt-app; Scheduler"
author: "By Jonathan Corbet September 18, 2017 Linux Plumbers Conference"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 18, 2017

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2017-Linux_Plumbers_Conference>)

The [scheduler workloads microconference](<https://www.linuxplumbersconf.org/2017/ocw/events/LPC2017/tracks/638>) at the 2017 Linux Plumbers Conference covered several aspects of the kernel's CPU scheduler. While workloads were on the agenda, so were a rework of the realtime scheduler's push/pull mechanism, a distinctly different approach to multi-core scheduling, and the use of tracing for workload simulation and analysis. As the following summary shows, CPU scheduling has not yet reached a point where all of the important questions have been answered. 

#### Workload simulation

First up was Juri Lelli, who talked about the [rt-app](<https://github.com/scheduler-tools/rt-app>) tool that tests the scheduler by simulating specific types of workloads. Work on rt-app has been progressing; recent additions include the ability to model mutexes, condition variables, memory-mapped I/O, periodic workloads, and more. There is a new JSON grammar that can be used to specify each task's behavior. Rt-app now generates a log file for each task, making it easy to extract statistics on what happened during a test run; for example, it can be used to watch how the CPU-frequency governor reacts when a "small" task suddenly starts requiring a lot more CPU time. Developers have been using it to see how scheduler patches change the behavior of the system; it is a sort of unit test for the scheduler. 

In summary, he said, rt-app has become a flexible tool for the simulation of many types of workloads. The actual workloads are generated from traces taken when running the use cases of interest. It is possible (and done), he said, to compare the simulated runs with the original traces. The [![\[Juri Lelli\]](https://static.lwn.net/images/conf/2017/ossna-lpc/JuriLelli-sm.jpg)](<https://lwn.net/Articles/734045/>) rt-app repository itself has a set of relatively simple workloads, modeling browser and audio playback workloads, for example. New workloads are usually created when somebody finds something that doesn't work well and wants a way to reproduce the problem. 

Rafael Wysocki said that he often has trouble with kernel changes that affect CPU-frequency behavior indirectly. He would like to have a way to evaluate patches for this kind of unwanted side effect, preferably while patches are sitting in linux-next at the latest and are relatively easy to fix. Might it be possible to use rt-app for this kind of testing? Josef Bacik added that his group (at Facebook) is constantly fixing issues that come up with each new kernel; it gets tiresome, so he, too, would like to find and fix these problems earlier. 

He went on to state that everybody seems to agree that rt-app is the tool for this job. There would be value in having all developers pool their workloads into a comprehensive battery of tests, but where is the right place to put them? The rt-app project itself isn't necessarily the right place for all these workloads, so it would seem that a different repository is indicated. The [LISA](<https://github.com/ARM-software/lisa>) project was suggested as one possible home. Steve Rostedt, however, suggested that these workloads could just go into the kernel tree directly. Lelli asked whether having rt-app as an external dependency would be acceptable; Rostedt said that it would. 

Wysocki wondered about how the community could ensure that these workloads get run on a wide variety of hardware; it's not possible to emulate everything. Bacik replied that it's not possible to test everything either, and that attempts to do so would be a case of the perfect being the enemy of the good. If each interested developer runs the tests on their own system, the overall coverage will be good enough. It's an approach that works out well for the xfstests filesystem-testing suite, he said. 

Peter Zijlstra complained that rt-app doesn't produce any direct feedback — there is no result saying whether the right thing happened or not. As a result, interpreting rt-app output "requires having a brain". Rostedt suggested adding a feature to compare runs against previous output and look for regressions. Patrick Bellasi noted, though, that to work well in this mode, the tests need to be fully reproducible; that requires care in setting up the tests. At this point, he said, rt-app is not a continuous-integration tool, but it could maybe be evolved in that direction. 

#### Reworking push/pull

Rostedt gave a brief presentation on what he called a "first-world problem". When running realtime workloads on systems with a large number (over 60) of CPUs — something he said people actually want to do — realtime tasks do not always migrate between CPUs in an optimal manner. That is due to shortcomings in how that migration is handled. 

When the last running realtime task on any given CPU goes idle, he said, the CPU will examine the system to see if any other CPUs are overloaded with realtime tasks. If it finds a CPU running more than one realtime process, it will pull one of those processes over and run it locally. Once upon a time, this examination was done by grabbing locks on the remote CPUs, but that does not scale well. Now, instead, the idle CPU will send an inter-processor interrupt (IPI) to tell other CPUs that they can push an extra realtime task in its direction. 

That is an improvement, but imagine a situation with many CPUs, one of which is overloaded. All of the other CPUs go idle more-or-less simultaneously, which does happen at times. They will all send IPIs to the busy CPU. One of them will get the extra process from that CPU but, having pushed that process away, the CPU still has to process a pile of useless IPIs. That adds extra load to the one CPU in the system that was already known to be overloaded, leading to observed high latencies — the one thing a realtime system is supposed to avoid at all costs. 

The proposed solution is to have only the first (for some definition of "first") idle CPU send the IPI. That IPI can then be forwarded on to other overloaded CPUs if needed. The result would be the elimination of the IPI storms. Nobody seemed to think that this was a bad solution. It was noted that it would be nice to have statistics indicating how well this mechanism is working, and the conversation devolved quickly into a discussion of tracepoints (or the lack thereof) in the scheduler code. Zijlstra said that he broke user space once as a result of tracepoints, so he will not allow the addition of any more. This is a topic that was to return toward the end of the session. 

#### Multi-core scheduling

Things took a different turn when Jean-Pierre Lozi stood up to talk about multi-core scheduling issues. Lozi is the lead author of the paper called [The Linux Scheduler: a Decade of Wasted Cores [PDF]](<https://www.ece.ubc.ca/~sasha/papers/eurosys16-final29.pdf>), which described a number of issues with the CPU scheduler. An attempt to go through a list of those bugs drew a strong response from Zijlstra, who claimed that most of them were fixed some time ago. 

The biggest potential exception is "work conservation" — ensuring that no task languishes in a CPU run queue if there are idle CPUs available elsewhere in the system. On larger systems, CPUs will indeed sit idle while tasks wait, and Zijlstra said that things will stay that way. When there are a lot of cores, he said, ensuring perfect work conservation is unacceptably expensive; it requires the maintenance of a global state and simply does not scale. Any cure would be worse than the disease. 

Lozi's proposed solution is partitioning the system, essentially turning it into a set of smaller systems that can be scheduled independently. Zijlstra expressed skepticism that such an idea would be accepted, leading to a suggestion from Lozi that this work may not be intended for the mainline kernel. There was a quick and contentious discussion on the wisdom of the idea and whether it could already be done using the existing CPU-isolation mechanisms. Mathieu Desnoyers eventually intervened to pull the discussion back to its original focus: a proposal to allow the creation of custom schedulers in the kernel. This work is based on [Bossa](<http://bossa.lip6.fr/>), which was originally developed some ten years ago. It uses a domain-specific language to describe a scheduler and how its decisions will be made; different partitions in the system could then run different scheduling algorithms adapted to their specific tasks. 

It was pointed out that the idea of enabling loadable schedulers was firmly shot down quite a few years ago. Even so, there was a brief discussion on how they could be enabled. The likely approach would be to create a new scheduler class that would sit below the realtime classes on the priority scale, but above the `SCHED_OTHER` class where most work runs. The discussion ran out of time, but it seems likely that this idea will eventually return; stocking up on popcorn for that event might be advisable. Zijlstra, meanwhile, insists that the incident with the throwable microphone was entirely accidental. 

#### Workload analysis with tracing

Desnoyers started the final topic of the session by stating that there is a real need for better instrumentation of the scheduler so that its decisions can be understood and improved. He would like exact information on process switches, wakeups, priority changes, etc. It is important to find a way to add this information without creating ABI issues that could impede future scheduler development. That means that the events have to be created in a way that allows them to evolve without breaking user-space tools. 

Zijlstra said that it would not be possible to add all of the desired information even to the existing tracepoints; that would bloat them too [![\[Mathieu Desnoyers\]](https://static.lwn.net/images/conf/2017/ossna-lpc/MathieuDesnoyers-sm.jpg)](<https://lwn.net/Articles/734046/>) much. Desnoyers suggested adding a new `version` file to each tracepoint; an application could write "`2`" to it to get a new, more complete output format. Rostedt complained about the use of version numbers, though, and suggested writing a descriptive string to the existing format file instead. Zijlstra said that tracepoints should default to the newest format, but Rostedt said that would break existing tools. The only workable way is to default to the old format and allow aware tools to request a change. 

That was about the point where Zijlstra (semi-jokingly) declared his intent to remove all of the tracepoints from the scheduler. "I didn't want this pony", he said. 

There was a wandering discussion on how it might be possible to design a mechanism that resembles tracepoints, but which is not subject to the kernel's normal ABI guarantees. Bacik said that a lot of the existing scripts in this area use kprobes; they are simply updated whenever they break. Perhaps a new sort of "tracehook" could be defined that resembles a kprobe: it is a place to hook into a given spot in the code, but without any sort of predefined output. A function could be attached to that hook to create tracepoint-style output; it could be located in a separate kernel module or be loaded as an eBPF script. Either way, that glue code would be treated like an ordinary kernel module; it is using an internal API and, as a result, must adapt if that API changes. 

The developers in the room seemed to like this idea, suggesting even that it might be a way to finally get some tracepoints into the virtual filesystem layer — a part of the kernel that does not allow tracepoints at all. Your editor was not convinced, though, that the ABI issues could be so easily dodged. As the session ended, it was resolved that the idea would be presented to Linus Torvalds for a definitive opinion, most likely during the Maintainers Summit in October. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, for supporting his travel to LPC 2017].
