---
title: The state of system observability with BPF
url: https://lwn.net/Articles/787131/
date: "May 1, 2019"
category: "BPF-Tracing"
author: "By Jonathan Corbet May 1, 2019 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 1, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The 2019 version of the Linux Storage, Filesystem, and Memory-Management Summit opened with a plenary talk by Brendan Gregg on observing the state of Linux systems using BPF. It is, he said, an exciting time; the BPF-based "superpowers" being added to the kernel are growing in capability and maturity. It is now possible to ask many questions about what is happening in a production Linux system without the need for kernel modifications or even basic debugging information. 

Gregg started with a demonstration tool that he had just written: it's immediate manifestation was in the creation of a high-pitched tone that varied in frequency as he walked around the lectern. It was, it turns out, a BPF-based tool that extracts the signal strength of the laptop's WiFi connection from the kernel and creates a noise in response. As he interfered with that signal with his body, the strength (and thus the pitch of the tone) varied. By tethering the laptop to his phone, he used the tool to measure how close he was to the laptop. It may not be the most practical tool, but it did demonstrate how BPF can be used to do unexpected things. 

Gregg works at Netflix, a company that typically operates about 150,000 server instances. Naturally, Netflix cares a lot about performance; that leads to a desire for observability tools that can help to pinpoint the source of performance problems. But the value of good tools goes beyond just performance tuning. 

#### Better bug reports

For example, the company is currently trying to move its server images forward to the 4.15 kernel, but a snag has come up. There is a log-rotation service that waits for a specific process to exit by repeatedly running `ps`. It seems like a simple task, but it started failing on the newer kernel; `ps` would fail to report the process when, in fact, that process was still running. It was, Gregg said, time to investigate. 

He began with the [`bpftrace`](<https://github.com/iovisor/bpftrace>) tool which, he said, is soon to be packaged by a number of distributors. It could be interesting, he thought, to see which system calls are failing under the new kernel that worked under the old. Finding out which system calls are failing can be done with a command like: 

```
bpftrace -e 't:syscalls:sys_exit_* /comm == "ps"/ \
        	     { @[probe, args->ret > 0 ? 0 : - args->ret] = count(); }'
```

This has to attach to all 316 system calls, so it takes a while to get going; there is talk of a "multi-attach" functionality to speed things up in the future. In the meantime, one can get around that problem by changing the command to attach to the raw system-call tracepoint instead: 

```
bpftrace -e 't:raw_syscalls:sys_exit /comm == "ps"/ \
        	      {@[args->id, args->ret > 0 ? 0 : - args->ret] = count(); } '
```

Gregg posted a lot of code in this session, most of which has not been reproduced here. See [his slides](<http://www.brendangregg.com/Slides/LSFMM2019_BPF_Observability.pdf>) for the full details. 

That works more quickly, at the cost of reporting system-call numbers rather than names. This command did, indeed, turn up one extra failure that was happening under the new kernel, but it turned out to have nothing to do with the problem — a dead end. So it was time to shift tactics: perhaps some system call was "successfully failing". A couple more `bpftrace` commands later, he had an answer: occasionally the `getdents()` system call will return a partial result on `/proc`, causing the entry for the target process to be left out. 

So, perhaps, the problem lives in the `/proc` filesystem. Switching to an ftrace-based tool for a moment, he looked at which functions were being called in the failed runs. Then back to `bpftrace` to chase a few wrong leads before determining that [`find_ge_pid()`](<https://elixir.bootlin.com/linux/latest/source/kernel/pid.c#L449>), which is supposed to continue a read by finding the next highest process ID, was stopping early. That, in turn, appears to have been caused by a change in the implementation of the process-ID table. And at that point he had to stop to travel to the conference. 

He thus didn't yet have a solution to the problem (though, since the guilty parties were in the room, a solution seems likely to come soon). But, he said, even at this point BPF has helped to significantly narrow down the problem. One of the biggest benefits to the current crop of BPF-based observability tools, he said, is that they enable the creation of much better bug reports. 

#### One-liners

Stepping back briefly, Gregg talked about the architecture of the BPF system. At the lowest level is the BPF virtual machine itself. One can write raw BPF code for this machine, but it's "super hard" and nobody should do it. He asked the people in the audience who had done this to raise their hands and was surprised by how many hands went up; the room contained, he said, the entire world population of raw BPF experts. For those wanting a higher-level experience, though, an LLVM backend allows BPF programs to be written in a restricted version of C. Then, the BCC system allows those programs to be loaded and driven from C or Python code. And now, above that, is `bpftrace`, which allows the writing of useful one-line tools. 

So, for example, he asked about the effectiveness of readahead in the page cache: is it actually accelerating I/O, or is it just polluting the cache? One need not wonder about such things; they can now be observed. So he wrote a simple tool in `bpftrace` to record timestamps for all pages brought in via readahead; it then compares the timestamps when those pages are actually [![\[Brendan Gregg\]](https://static.lwn.net/images/conf/2019/lsfmm/BrendanGregg-sm.jpg)](<https://lwn.net/Articles/787181/>) referenced. The result, for his workload, was that almost all pages were accessed within 32ms of being read into the cache; the number of pages that were never referenced was quite small. So readahead is indeed helping here; if anything, the application in question could benefit from even more readahead. 

One can ask similar questions about eviction from the page cache: by looking at the age of pages as they are pushed out, it is possible to evaluate whether the page cache is appropriately sized. If those pages have not been referenced in a long time, perhaps the cache is too large. This tool does not yet exist but, he suggested, perhaps the assembled group could put an implementation together during the conference and send it to him. 

The tool to answer the readahead question was not quite a one-liner, but it fit nicely onto a single slide. `bpftrace`, he said, is a great platform for the creation of short tools. In its simplest form, one need only provide a probe (a kernel tracepoint, for example), an optional filter expression, and an action to take when the probe is hit. The language overall, he said, looks a lot like awk or DTrace. 

Inside, it has a parser that builds an abstract syntax tree describing the task, which is then used to create the BPF program(s) to attach to the events of interest. The perf ring buffer is used to get high-bandwidth data out of the kernel but, to the greatest extent possible, data is boiled down in kernel space first and exported via BPF maps. Version 0.90 was released in March; the next release, called 0.10, is due soon (though that release number may be changed after some complaints were raised about the apparent backward number jump). 

#### BPF everywhere

As Gregg works on a book about performance analysis with BPF, he is trying to fill in various "observability gaps" in the system. To that end, he brought back his old `superping` utility, which hooked into packet send and receive events in an attempt to obtain more accurate ping times by eliminating any scheduling latency. But he ended up reporting longer times than ordinary `ping` does — not the expected result. It turns out that this problem has been solved in the networking stack years ago, with timestamps being stored in the packets themselves, so this tool proved unnecessary. The `superping` tool demonstrates, though, how BPF programs can work from kernel headers to walk through chains of structures. 

A tool that is still useful, instead, is called [`tcplife`](<http://www.brendangregg.com/blog/2016-11-30/linux-bcc-tcplife.html>); it collects information about TCP connections to and from the local system. This kind of data is normally generated through packet capture, which is an expensive thing to do; `tcplife`, instead, just hooks into the networking stack to get events when connections begin and end. Initially it used a kprobe set on `tcp_set_state()`; that information proved useful enough that a proper tracepoint was added in 4.15. That was an improvement; tracepoints are more stable and less prone to breaking than kprobes. But only to a point. 

In 4.16, that tracepoint was moved to `inet_sock_set_state()` and some more information was added to it; that broke all of Gregg's tools that were using the older tracepoint. And, he said, he is "fine with that". Even if a tracepoint changes occasionally, it's far better than using raw kprobes. He did have one request, though: tracepoints should include a direct pointer to the structure(s) in use at that location in the code. He realizes that any code using that pointer will be unstable, but it's still useful to have if there is a need for data that is not exported directly by the tracepoint. 

This case illustrates a more general point, he said: kprobes can serve as a useful way to prototype tracepoints. A tool built around a kprobe shows what the use case for the tracepoint would be and which data should be included; that helps to justify the tracepoint's addition and to get it right when that happens. 

Gregg concluded with a "reality check": even though the various BPF tools provide all kinds of useful information, most performance wins still come from poring over flame graphs instead. But, naturally, BPF is moving into that area as well; it is now possible to create trace summaries in the kernel, reducing the overhead of collecting that sort of performance data. That, he said, shows the direction the kernel community should continue to take in the future: "BPF all the things".
