---
title: Lots of new perf features
url: https://lwn.net/Articles/593690/
date: "April 9, 2014"
category: Performance monitoring
author: "By Jake Edge April 9, 2014 Collaboration 
Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
April 9, 2014

* * *

[Collaboration Summit](<https://lwn.net/Archives/ConferenceByYear/#2014-Collaboration_Summit>)

New features for the perf tracing tool and its infrastructure were the topic of two talks squeezed into one slot at the 2014 Linux Foundation Collaboration Summit. Red Hat's Jiri Olsa and LG Electronics's Namhyung Kim looked at both recent changes that have gone into perf as well as those that are still pending. Each looked at a different set of improvements to perf, including those they created themselves and those that came from other developers. 

[ ![\[Jiri Olsa\]](https://static.lwn.net/images/2014/lfcs-olsa-sm.jpg) ](<https://lwn.net/Articles/593909/>)

Olsa started things off by describing the `libtraceevent` plugins that will help in parsing the tracepoint format lines. `libtraceevent` itself was written by Steven Rostedt as part of the trace-cmd front-end for ftrace, but is now used by perf as well. Each kernel tracepoint has formatting information in its `format` file ("print fmt") that describes the format used by the kernel to output the data from the tracepoint, but that line is difficult for user space code to use, he said. Plugins will allow user space to easily parse that information to produce the same output as the kernel. 

#### Intel processor feature support

Support for the "running average power limit" (RAPL) feature in Intel processors was next up. It allows administrators to set and monitor power consumption limits for various hardware domains in the system. With this addition, authored by Stephane Eranian, perf can sample and report on power consumption in several different categories: all physical cores, the processor package, the DRAM, or the built-in GPU. 

Another Intel-only feature allows perf to do memory profiling on those systems. Recent Intel CPUs provide a way to see the individual loads and stores of memory. Each of those events has additional information associated with it, including the instruction address, data address, location of access (L1 cache, local RAM, or remote cache, for example), and type of translation lookaside buffer (TLB) access (hit, miss, L1, ...). There is also information on the "weight" of the access, which is the amount of work in cycles that the processor spent on the memory access. The weight values are not particularly reliable, Olsa said, but are getting better with each new processor. 

Perf itself only records the memory access information, it does not try to analyze it. Another tool, c2c (for "cache to cache"), tries to detect cache-line sharing between processors, which is a good thing to avoid, he said. It will report on each cache line, giving the address of the line, the type of access (load/store), the offset of the access, and the instruction that caused the access. With that information, developers should be able to spot cache-line bouncing in their systems. 

#### Better backtrace

Improvements to the backtrace output from ftrace was next on Olsa's list. The idea is to be able to see the call chain that led up to a particular sample. That is currently done using frame pointers. If frame pointers are not available, the newly added `libdw` can use the DWARF debugging data format to unwind the stack and produce a backtrace. Perf can record the user stack and registers, then use `libdw` to provide a backtrace at report time, which is faster than using `libunwind` as was done previously. 

An even faster mechanism to produce backtraces uses the "last branches record" (LBR) information. Enabling LBR will store a list of the last branches taken by the code each time a sample is taken. Both the from and to addresses are stored and that information can be used to produce a backtrace. 

#### Future changes

All of the preceding changes have gone into perf relatively recently; Olsa then turned to some features that can be expected in the future. There is a plan to support the [common trace format](<http://www.efficios.com/ctf>) (CTF) in perf; the other user of CTF currently is the Linux trace toolkit next generation (LTTng). Making "perf record" multi-threaded is also on the drawing board. That will require per-CPU storage for perf data. Supporting multiple output data files (with a maximum size per file) is planned as well. 

Some bigger features that are coming include a mechanism to allow events to toggle other events on or off to help narrow down the measured area. The original code was from Frédéric Weisbecker, but it is missing a suitable user interface, Olsa said. Currently it has an "ugly" command-line interface. 

Currently running "perf record" requires an open file for each CPU for each event. For traces with lots of events on systems with lots of CPUs, the maximum open file descriptor limit will be hit. So there are plans to allow for "event groups" that would share a single file descriptor. This idea has only been discussed, Olsa said, no code has been posted. 

Generic code will be moved out of perf itself and into the kernel `tools/lib` directory so that other tools have access to it. In closing, Olsa mentioned that there are 26 or so automated tests for perf that get run each time a commit is made. That test suite is getting bigger over time. 

#### Output changes

[ ![\[Namhyung Kim\]](https://static.lwn.net/images/2014/lfcs-kim-sm.jpg) ](<https://lwn.net/Articles/593910/>) With that, Kim took over to discuss even more perf enhancements. A change to "perf report" will allow users to see the caller information more clearly, he said. The `--children` option will show the full call chain and report each member of the call chain's impact on the measured quantity. It adds "Children" and "Self" columns to the perf output showing the cumulative contribution at each step of the call chain. 

Another output change is the `--fields` option, which specifies a comma-separated list of the columns that will be in the output. It works with the existing `--sort` option, allowing users to specify which columns to sort and which to output. 

#### Ftrace support

Integrating ftrace support into perf is another new feature. The idea is to integrate the tool side of ftrace into perf, Kim said. Currently, only the function and function_graph tracers are supported. The feature is using the `libtraceevents` kbuffer API to access the events. Providing perf-like behavior using ftrace events is the goal. 

To use it, one does a `perf ftrace record`. After the recording, one can get ftrace-like output using `perf ftrace show` or perf-like output using `perf ftrace report`. It essentially allows the user to access the ftrace ring buffer and its events from perf. More tracers will be added over time. 

DWARF support has been added to uprobes, which allows perf to place dynamic tracepoints using symbolic names and line numbers. That work was done by Masami Hiramatsu and is already upstream, Kim said. Using the `--line` and `--vars` options will use the debug information in a binary to create uprobe tracepoints that perf can sample for those locations. 

#### Statically defined tracepoints

The final new feature that Kim covered was statically defined tracepoints (SDTs), which are similar to the kernel's tracepoints, but specified for user-space applications. SDT is the same format used by DTrace and multiple applications have already added tracepoints in that format. With the addition of SDT support, perf can access and sample these SDTs. There is support for listing which SDTs are available in a program and support for processing them using uprobes. Support for SDT arguments and for treating SDTs like normal perf events is coming, Kim said. 

As can be seen, there is a lot of activity going on in the perf world. Much of it is straightforward improvements as well as support for new processor features, but the ftrace integration and event toggling will be fairly substantial new pieces. Making it all work with user-space programs and DTrace tracepoints are also nice benefits. For Linux tracing, most of the facilities are now in place, but the interfaces are still geared toward advanced developers and kernel hackers—simpler interfaces would seem to be an area that needs more attention in the future. 

[ Thanks to the Linux Foundation for supporting my travel to the Collaboration Summit. ]
