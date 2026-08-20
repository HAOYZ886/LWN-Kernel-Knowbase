---
title: Using the perf code to create a RAS daemon
url: https://lwn.net/Articles/425990/
date: "February 2, 2011"
category: Performance monitoring
author: "By Jake Edge February 2, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
February 2, 2011

Monitoring a system for "reliability, availability, and serviceability" (RAS) is an important part of keeping that system, or a cluster of such computers, up and running. There is a wide variety of things that could be monitored for RAS purposes--memory errors, CPU temperature, RAID and filesystem health, and so on--but Borislav Petkov's [RAS daemon](<https://lwn.net/Articles/424071/>) is just targeted at gathering information on any machine check exceptions (MCEs) that occur. The daemon uses trace events and the perf infrastructure, which requires a fair amount of restructuring of that code to make it available not only to the RAS daemon, but also for other kinds of tools. 

The first step is to create persistent perf events, which are events that are always enabled, and will have event buffers allocated, even if there is no process currently looking at the data. That allows the MCE trace event to be enabled at boot time, before there is any task monitoring the perf buffer. Once the boot has completed, the RAS daemon (or some other program) can `mmap()` the event buffers and start monitoring the event. This will allow the RAS daemon to pick up any MCE events that happened during the boot process. 

To do that, the `struct perf_event_attr` gets a new `persistent` bitfield that is used to determine whether or not to destroy the event buffers when they are unmapped. In addition, persistent events can be shared by multiple monitoring programs because they can be mapped as shared and read-only. Once the persistent events are added, the next patch then changes the MCE event to become a persistent event. 

With that stage set, Petkov then starts to rearrange the perf code so that the RAS daemon and other tools can access some of the code that is currently buried in the `tools/perf` directory. That includes things like the trace event utilities, which move from `tools/perf/util` to `tools/lib/trace` and some helper functions for debugfs that move to `tools/lib/lk`. These were obviously things that were needed when creating the RAS daemon, but not easily accessible. 

A similar patch moves the `mmap()` helper functions from the `tools/perf` directory to another new library: `tools/lib/perf`. These functions handle things like reading the head of the event buffer queue, writing at the tail of the queue, and reading and summing all of the per-cpu event counters for a given event. 

In [response](<https://lwn.net/Articles/426027/>) to the patch moving the `mmap()` helpers, Arnaldo Carvalho de Melo pointed out that he had already done some work to rework that code, and that it would reduce the size of Petkov's patch set once it gets merged into the -tip tree. He also noted that he had created a set of Python bindings and a simple perf-event-consuming `twatch` daemon using those bindings. While Petkov had [some reasons](<https://lwn.net/Articles/426030/>) for writing the RAS daemon in C rather than Python, mostly so that it would work on systems without Python or with outdated versions, he did seem impressed: ""twatch looks almost like a child's play and even my grandma can profile her system now :)."" 

But the Python bindings aren't necessarily meant for production code, as Carvalho de Melo [describes](<https://lwn.net/Articles/426032/>). Because the Python bindings are quite similar to their C counterparts, they can be used to ensure that the kernel interfaces are right: 

I.e. one can go on introducing the kernel interfaces and testing them using python, where you can, for instance, from the python interpreter command line, create counters, read its values, i.e. test the kernel stuff quickly and easily. 

Moving to a C version then becomes easy after the testing phase is over and the kernel bits are set in stone. 

There are some additional patches that move things around within the `tools` tree before the final patch actually adds the RAS daemon. The daemon is fairly straightforward, with the bulk of it being boilerplate daemon-izing code. The rest parses the MCE event format (from `mce/mce_record/format` file in debugfs), then opens and maps the debugfs `mce/mce_recordN` files (where N is the CPU number). The main program sits in a loop checking for MCE events every 30 seconds, printing the CPU, MCE status, and address for any events that have occurred to a log file. Petkov mentions decoding of the MCE status as something that he is currently working on. 

Obviously, the RAS daemon itself is not the end result Petkov is aiming for. Rather, it is just a proof-of-concept for persistent events and demonstrates one way to rearrange the perf code so that other tools can use it. There may be disagreements about the way the libraries were arranged, or the specific location of various helpers, but the overall goal seems like a good one. Whether tools like `ras` actually end up in the kernel tree is, perhaps, questionable--the kernel hackers may not want to maintain a bunch of tools of this kind--but by making the utility code more accessible, it will make it much easier for others build these tools on their own.
