---
title: Tracepoints with BPF
url: https://lwn.net/Articles/683504/
date: "April 13, 2016"
category: "BPF-Tracing; Tracing-with BPF"
author: "By Jonathan Corbet April 13, 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 13, 2016

One of the attractive features of tracing tools like SystemTap or DTrace is the ability to load code into the kernel to perform first-level analysis on the trace data stream. Tracing can produce vast amounts of data, but that data can often be reduced considerably by some simple processing — incrementing histogram buckets, for example. Current kernels have a wealth of tracepoints, but they lack the ability to perform arbitrary processing of trace events in kernel space before exporting the result. It would appear, though, that this situation will change as the result of a set of patches targeted for the 4.7 release. 

It should come as no surprise to regular LWN readers at this point that the technology being used for the loading of code into the kernel is the BPF virtual machine. BPF allows code to be executed in kernel space under tight constraints; among other things, it can only access data that is explicitly provided to it and it cannot contain loops; thus, it is guaranteed to run within a bounded time. BPF code can also be translated to native code with the in-kernel just-in-time compiler, making it fast to run. This combination of attributes has helped BPF to move beyond the networking stack and make inroads into a number of kernel subsystems. 

Every BPF program loaded into the kernel has a specific type assigned to it; that type restricts the places where the program may be run. [The patch set](<https://lwn.net/Articles/682936/>) from Alexei Starovoitov creates a new type (`BPF_PROG_TYPE_TRACEPOINT`) for programs intended to be attached to tracepoints. Those programs can then be loaded into the kernel with [the `bpf()` system call](<https://lwn.net/Articles/612878/>). Actually attaching a program to a tracepoint is done by opening the tracepoint file (in debugfs or tracefs), reading the tracepoint ID, then using the `PERF_EVENT_IOC_SET_BPF` `ioctl()` command. That command exists in current kernels to allow BPF programs to be attached to kprobes; the patch set extends it to do the right thing depending on the type of BPF program passed to it. 

When a tracepoint with a BPF program attached to it fires, that program will be run. The "context" area passed to the program is simply the tracepoint data as it would be passed to user space, except that the "common" fields are not accessible. As an example, the patch set includes a sample that attaches to the `sched/sched_switch` tracepoint, which fires when the scheduler switches execution from one process to another. The `format` file for that tracepoint (found in the tracepoint directory in debugfs or tracefs) provides the following data: 

```
field:unsigned short common_type;	offset:0;	size:2;	signed:0;
        field:unsigned char common_flags;	offset:2;	size:1;	signed:0;
        field:unsigned char common_preempt_count;	offset:3;	size:1;signed:0;
        field:int common_pid;	offset:4;	size:4;	signed:1;
    
        field:char prev_comm[16];	offset:8;	size:16;	signed:1;
        field:pid_t prev_pid;	offset:24;	size:4;	signed:1;
        field:int prev_prio;	offset:28;	size:4;	signed:1;
        field:long prev_state;	offset:32;	size:8;	signed:1;
        field:char next_comm[16];	offset:40;	size:16;	signed:1;
        field:pid_t next_pid;	offset:56;	size:4;	signed:1;
        field:int next_prio;	offset:60;	size:4;	signed:1;
```

Any program that accesses tracepoint data is expected to read this file to figure out which data is available and where it is to be found; failure to do so risks trouble in the future should the data associated with this tracepoint change. An in-kernel BPF program cannot read this file, so another solution must be found. That solution is for the developer to read the `format` file and turn it into a C structure; there is a tool (called `[tplist](<https://github.com/iovisor/bcc/blob/master/tools/tplist.py>)`) that will do this job. The patch set contains the following structure, which was generated with `tplist`: 

```
/* taken from /sys/kernel/debug/tracing/events/sched/sched_switch/format */
        struct sched_switch_args {
    	unsigned long long pad;
    	char prev_comm[16];
    	int prev_pid;
    	int prev_prio;
    	long long prev_state;
    	char next_comm[16];
    	int next_pid;
    	int next_prio;
        };
```

The `pad` field exists because the first four fields (those common to all tracepoints) are not accessible to BPF programs. The rest, however, can be accessed by name in a C program (which will be compiled to BPF and loaded into the kernel). This program will likely extract the data of interest from this structure, process it in its own special way, and store the result in a BPF map; user space can then access the result. 

As with the other BPF program types, the helper code supplied with the kernel uses section names as a directive for what should be done with a specific program. So a program meant to be attached to a tracepoint should be explicitly placed in a section called "`tracepoint/_name_` ", where "`_name_` " is the name of the tracepoint of interest. So, for the sample program, the section name is "`tracepoint/sched/sched_switch`". 

The mechanism works, and, importantly, a tracepoint-attached BPF program is quite a bit more efficient than placing a kprobe and attaching a program there. There are already tools in development ([`argdist`](<https://github.com/iovisor/bcc/blob/master/tools/argdist.py>), for example) that will create BPF programs for specific tasks; `argdist` will create a program to make a histogram of the values of a given tracepoint field. All told, it looks like a useful advance in the kernel's instrumentation. 

There is a potential catch, though: the old issue of tracepoints and ABI stability. Tracepoints expose the inner workings of the kernel, which suggests that they must change if the kernel does. Changing tracepoints can, however, break applications that use them; this is an issue [that has come up many times](<https://lwn.net/Articles/442113/>) in the past. It is also why certain subsystems (the virtual filesystem layer, for example) do not allow tracepoints at all: the maintainers are worried that they might be unable to make important changes because they may break applications dependent on those tracepoints. 

For user-space programs, the issue has been mitigated somewhat by providing library code to access tracepoint data. An application that uses these utilities should be portable across multiple kernel versions. BPF programs, though, do not have access to such libraries and will break, perhaps silently, if the tracepoints they are using are changed. ABI concerns have stalled the merging of this capability in the past, but there was little discussion of ABI worries this time around. Alexei maintains that the interface available to BPF programs is the same as that seen by user-space programs, so there should be no new ABI worries. Whether the BPF interface truly brings no new ABI issues is something that will have to be seen over the coming years. 

And it does appear that we will have the chance to see how that plays out; David Miller has [applied the patches to the net-next tree](<https://lwn.net/Articles/683534/>), meaning that they should reach the mainline in the 4.7 merge window. Users wanting more visibility into what's happening inside the kernel will likely be happy to have it.
