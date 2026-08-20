---
title: Unifying kernel tracing
url: https://lwn.net/Articles/803347/
date: "October 30, 2019"
category: "Development tools-Kernel tracing; Tracing"
author: "By Jake Edge October 30, 2019 OSS EU"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
October 30, 2019

* * *

[OSS EU](<https://lwn.net/Archives/ConferenceByYear/#2019-Open_Source_Summit_Europe>)

Steven Rostedt has been a part of the Linux kernel tracing community for most of its existence, it seems. He was the developer of [ftrace](<https://www.kernel.org/doc/Documentation/trace/ftrace.txt>), which was one of the early mainline additions for tracing. There are now many tracing facilities in the kernel. At the [2019 Open Source Summit Europe](<https://events19.linuxfoundation.org/events/open-source-summit-europe-2019/>) in Lyon, France, Rostedt wanted to present an idea that he has been thinking about for a long time: a unified tracing platform to provide access to all of the kernel tracing facilities from user-space applications. 

There are "many faces of tracing" for Linux, including [perf](<https://perf.wiki.kernel.org/index.php/Main_Page>), [LTTng](<https://lttng.org>), [SystemTap](<https://sourceware.org/systemtap/>), [DTrace](<http://dtrace.org/blogs/about/>), [BPF](<https://lwn.net/Articles/740157/>), [ktap](<https://github.com/ktap/ktap>), [strace](<https://strace.io>), [GDB](<https://www.gnu.org/software/gdb/>), ftrace, and "there's plenty more", he said. He began his presentation with a look at the history of Linux tracing, touching on a number of those projects. 

#### Tracing history

Ftrace actually started back when Rostedt was working on his Master's thesis in 1998. He created "logdev" to print messages to a ring buffer in a way that was faster than `printk()`. He was investigating network quality of service (QoS) behavior and `printk()` was simply too slow. He got a job at Timesys after that and needed a faster way to get information out of that company's realtime kernel; he was easily able to get logdev working on that kernel. His first version of logdev was for the 2.0 kernel; he ported it to 2.2, 2.4, and so on as they became available. He actually used logdev to debug ftrace as he was developing it; logdev is one of the two parents of ftrace. 

[ ![\[Steven Rostedt\]](https://static.lwn.net/images/2019/osseu-rostedt-sm.jpg) ](<https://lwn.net/Articles/803395/>)

The realtime patch set had a latency tracer that is the other parent. It was created in 2004 by Ingo Molnar and Nadia Chambers. The latency tracer used a GCC profiling feature that would call an `mcount()` routine at the beginning of every function, which could be used to trace the execution of the kernel. The latency tracer was not dynamic, as function tracing could not be controlled at runtime; it was either compiled in or not. 

In 2008, Rostedt was at Red Hat on the realtime team, which was working on getting the realtime patches upstream. Arnaldo Carvalho de Melo tried to port the latency tracer to the upstream kernel, but the tracer was quite complex and hard to follow; it called `trace()`, which called `_trace()` and so on up to `_____trace()` (five underscores). Rostedt was familiar with the code, so he cleaned it up, overhauled the latency tracer to use logdev, and turned all of that into ftrace. 

Next came perf, which started as a "profiling utility done correctly"; it came out of the flame wars that surrounded oprofile and other profiling solutions, he said. In addition to profiling, perf had some tracing features that it got by hooking into the ftrace infrastructure. Molnar wanted Rostedt to stop developing ftrace and to move all of that effort into perf, but Rostedt resisted that push in part because he wanted to keep ftrace's debugfs interface. No user-space tool was needed to use ftrace via debugfs, which made it useful for embedded Linux systems that did not have build tools; just using `echo` from BusyBox was enough to enable tracing and a simple `cat` could show the tracing results. 

Before LTTng, there was the Linux Trace Toolkit (LTT) that was created around 2000 by Karim Yaghmour. It was the first real attempt to get a tracing solution into the mainline, but it ran aground on objections from Molnar and Linus Torvalds. 

The "next generation" (ng) of LTT was actually a complete rewrite done by Mathieu Desnoyers in 2006. Rostedt noted that Desnoyers has said that it was a mistake to keep LTT in the name since it shared little with that code base and was confusing to those who thought it was simply an extension of LTT. While LTTng never made it upstream and has been maintained over the years as an out-of-tree patch set, Desnoyers was able to get tracepoints into the mainline; the trace events used by ftrace are built on top of tracepoints. 

DTrace was the one of the first tracing tools to allow scripting in the kernel. It was originally for Solaris and was officially released in 2005. After Oracle acquired Sun, the company ported DTrace to Linux, but did not make it available under the GPL until 2017. That was too late for most, so DTrace is only available on Oracle Linux (and perhaps a few others). It is interesting to note that some say DTrace came from IBM's dprobe work, which was much earlier and shipped with SUSE Linux Enterprise Server (SLES) kernels at that time, Rostedt said. Everyone builds on each others' work in the grand tradition of "on the shoulders of giants". 

Red Hat's answer to DTrace was SystemTap, which was released in 2009. Like some of the other solutions, it never made it into the mainline kernel. Part of the problem with getting SystemTap, LTTng, and others upstream is that those projects focused on the tracer side but never sold other developers on the usefulness of those tools. 

Ftrace was not written as a tracing tool, per se, but it was part of the effort to get the realtime patches upstream, which needed the ftrace facilities for specific purposes. He worked with Desnoyers to get tracepoints into the kernel, then started making it easy for other developers to add trace events to their subsystems. The [macros used to create trace events](<https://elixir.bootlin.com/linux/v5.4-rc2/source/include/linux/tracepoint.h>) are horrifying to look at but easy to use. So ftrace got its foot in the door, then expanded as other kernel developers started to see its utility. Now there are over 1000 trace events in the mainline kernel. 

A developer could simply define a trace event using the macro and it would appear in the tracefs hierarchy. A simple `echo 1` into that file would enable the event, which would start showing up in the tracing output. Once kernel developers realized how easy it was, they started being much more interested in tracing and wanted to see more tracing features added. At the 2008 Kernel Summit, Torvalds [proclaimed](<https://lwn.net/Articles/298685/>) that SystemTap was far too complicated and that the simple tracing tools already in the kernel should be expanded further instead. Simplicity was the goal of the early tools, Rostedt said. 

Then came BPF. A just-in-time (JIT) compiler was added for BPF on x86 in 2011, which made customized network-packet filtering extremely fast. In 2014, Alexei Starovoitov introduced extended BPF (eBPF) that allowed BPF programs to be applied to other areas of the kernel, including tracing. Tracing with eBPF provides Linux with the capabilities that DTrace and SystemTap were targeting. And now, both DTrace and SystemTap are being reworked to use eBPF under the covers so that existing scripts for those tools will function on mainline kernels. 

#### Status

He referred to a 2017 [tweet from Julia Evans](<https://twitter.com/b0rk/status/881003937750544384>) that described the state of tracing on Linux, which she turned into a lengthy [blog post](<https://jvns.ca/blog/2017/07/05/linux-tracing-systems/>); all of that information is still pretty accurate today, Rostedt said. He is often asked "why can't we have just one tracer?" Some people seem to think that too much choice is a problem but his suggestion for those people is a switch to macOS 

Part of the complaint about having multiple tracers is that it is splitting the development effort, but he believes that "diversity brings innovation". If the Bell System had not been [broken up](<https://en.wikipedia.org/wiki/Breakup_of_the_Bell_System>) back in the 1980s, which brought choice and diversity to the telephone world, we might still all be using [rotary phones](<https://en.wikipedia.org/wiki/Rotary_dial>)—a device that many don't even recognize today. 

There is no "one size fits all" in tracing (or much of anything else); he pointed to "TABs versus spaces" and "Emacs versus Vim" as two classic examples. Tracing users often have different objectives. A dump truck could be used to travel between cities, but a sports car or motorcycle might be more appropriate; there are different needs at different times so it is nice to have a diversity of tools to address them. 

Some complain that diversity killed Unix, he said; there were too many flavors of Unix, which led to its demise. But those Unix flavors were all proprietary, so they could not share features. In the proprietary world, forks are bad, but in the open-source world, forks are good, he said. Those forks allow people to try things out in different, unexpected ways; if the outcome is useful, it can folded back into the original. "Diversity is the strength of open source", he said, "because we can always share". 

#### Commonalities

> [ ![\[Tracing commonality\]](https://static.lwn.net/images/2019/osseu-commonality-sm.png) ](<https://lwn.net/Articles/803450/>)

There are a number of pieces that are shared between the tracers. Tracepoints are being used by all of the different solutions, as are kernel probes (kprobes) and user-space probes (uprobes). The ftrace function hooks are also being used by many of the tracers. He showed the diagram above (from his [slides [PDF]](<https://static.sched.com/hosted_files/osseu19/5f/unified-tracing-platform-oss-eu-2019.pdf>)) to demonstrate the pieces that are shared within the world of Linux tracers. The top half, above the dotted line, contains the user-space pieces, while the lower part has all of the various kernel pieces. 

His idea is to have a single user-space library that can be used by all of the different tracers to interface to all of the different kernel facilities. It can be seen in the diagram below as the large green box in user space—with a bit of a typo in its label. Effectively, each of the tools would have access to the capabilities used by the others, so the dump truck, sports car, and motorcycle of the tracing world can "unleash the power of all the infrastructure that's in the kernel". 

> [ ![\[Unified tracing library\]](https://static.lwn.net/images/2019/osseu-unified-sm.png) ](<https://lwn.net/Articles/803453/>)

There has already been some progress in that direction, Rostedt said. [Babeltrace](<https://diamon.org/babeltrace/>) is a library that came out of the LTTng project. It can convert between multiple different tracing-file formats and the [Common Trace Format](<https://diamon.org/ctf/>) (CTF). The goal is to allow any tracing utility to read and use the data handled by any other. 

Another entry is libtraceevent, which is now ready to be packaged and distributed, Rostedt said; he is looking for distribution package maintainers to help out with that. It lives in the kernel source tree (in `tools/lib/traceevent`). When raw trace event data is written by the kernel, the format of that data is also exported in tracefs; that information is used by perf, [PowerTOP](<https://01.org/powertop/>), [trace-cmd](<http://man7.org/linux/man-pages/man1/trace-cmd.1.html>), and others. All of those tools copied the code that he wrote to parse the format information and access the data accordingly. He suggested that those projects copy the code, but now that it is available in library form, the code copies should be removed in favor of linking with libtraceevent. 

The [libperf](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/perf/lib>) library is available to wrap the [`perf_event_open()`](<http://man7.org/linux/man-pages/man2/perf_event_open.2.html>) system call. That call is effectively an `ioctl()` with "hundreds of commands" that is complex to use. Libperf provides a more sane interface to `perf_event_open()`. It is still a new project, but it will help applications gain access to the power underlying perf. 

There are also some libraries "coming soon", he said. Libftrace is an interface to the tracefs directory; it will allow users to start and stop tracers, enable and disable events, create kprobes and uprobes, and read the raw tracing data. Libtrace-cmd is a higher-level library that will allow other applications to do anything that trace-cmd can do. Similarly, libkshark is a GUI library that gives applications the ability to do anything that [KernelShark](<http://kernelshark.org/Documentation.html>) does. That means the capabilities of KernelShark are not limited to only handling ftrace data; other tracers can use the features of the GUI tool to work with their own data. 

That new box in the diagram would encompass these libraries and others to form a unified tracing platform for Linux. It would be an abstraction layer over all of the disparate capabilities that the Linux kernel provides for tracing. He suggested that the "Unix way" of "do one thing and do it well" is evolving into the "Linux way" of "do one library and do it well". 

The tracing tools are not competing with each other anymore, Rostedt said. There were the tracing wars of days gone by, which were good because they fostered innovation, but there is no reason to keep warring. The tracing projects can take the best ideas from each other and meld them into their tools in various ways, which might lead to electric sports cars, then electric motorcycles and dump trucks. He concluded by saying that the ability to do that kind of thing is what makes Linux the best operating system in the world. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to attend Open Source Summit Europe in Lyon, France.]
