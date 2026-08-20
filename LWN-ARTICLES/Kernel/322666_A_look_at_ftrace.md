---
title: A look at ftrace
url: https://lwn.net/Articles/322666/
date: "March 13, 2009"
category: Ftrace
author: "By Jake Edge March 13, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
March 13, 2009

There are quite a variety of [tracing options for Linux](<https://lwn.net/Articles/291091/>), with SystemTap being the most prominent, but that particular solution has yet to become easily usable, at least partly due to its many dependencies on user-space configuration and tools. Another choice, which originally came from work on realtime Linux, is ftrace. Ftrace is a self-contained solution, requiring no user-space tools or support, that is useful for tracking down problems--not only in the kernel, but in its interactions with user space as well. 

The name ftrace comes from "function tracer", which was its original purpose, but it can do more than that. Various additional tracers have been added to look at things like context switches, how long interrupts are disabled, how long it takes for high-priority tasks to run after they have been woken up, and so on. Its genesis in the realtime tree is evident in the tracers so far available, but ftrace also includes a plugin framework that allows new tracers to be added easily. 

In a suitably configured kernel--one with `CONFIG_FUNCTION_TRACER` enabled--accessing ftrace is done through the debug filesystem. Typically, it is mounted this way: 

```
# mount -t debugfs nodev /debug
```

That creates a `/debug/tracing` subdirectory which is used to control ftrace and for getting output from the tool. 

One chooses a particular tracer to use by writing to `/debug/tracing/current_tracer`--possibly after querying the tracers available by reading `/debug/tracing/available_tracers`. On a recent Fedora rawhide kernel, checking the tracers available results in: 

```
# cat /debug/tracing/available_tracers
        power wakeup irqsoff function sysprof sched_switch initcall nop
```

Tracing is then enabled or disabled by writing to `/debug/tracing/tracing_enabled`. Using the function tracer would be done as follows: 

```
# echo function >/debug/tracing/current_tracer
        # echo 1 >/debug/tracing/tracing_enabled
        ...some commands or activity to trace...
        # echo 0 >/debug/tracing/tracing_enabled
```

This produces a trace of each kernel function called, along with a timestamp, which will allow a kernel hacker to see what is going on inside the kernel. 

Output from ftrace can be read from one of several files in the `tracing` directory: 

  * `trace` \- contains human-readable output of the trace 

  * `latency_trace` \- output from the same trace, but organized so that system latency issues can be diagnosed, also in human-readable format 

  * `trace_pipe` \- contains the same output as `trace`, but is meant to be piped into a command. Unlike the other two, reading from this file consumes the output. 
The ring buffer used by ftrace is a fixed size (governed by the `buffer_size_kb` file), so `trace` or `latency_trace` entries will eventually be overwritten. 

Here are examples of the output from a function trace. The header helps to decode the various fields in the trace. This is what the output from `trace` looks like (heavily edited for brevity's sake): 

```
# tracer: function
        #
        #           TASK-PID    CPU#    TIMESTAMP  FUNCTION
        #              | |       |          |         |
    		bash-3330  [000]   147.799029: sys_open <-syscall_call
    		bash-3330  [000]   147.799030: do_sys_open <-sys_open
    		bash-3330  [000]   147.799030: getname <-do_sys_open
    		bash-3330  [000]   147.799031: kmem_cache_alloc <-getname
```

While this is `latency_trace` output (similarly edited): 

```
# tracer: function
        #
        function latency trace v1.1.5 on 2.6.29-0.215.rc7.fc11.i586
        --------------------------------------------------------------------
         latency: 0 us, #120119/5425477, CPU#0 | (M:desktop VP:0, KP:0, SP:0 HP:0 #P:2)
    	-----------------
    	| task: -0 (uid:0 nice:0 policy:0 rt_prio:0)
    	-----------------
    
        #                  _------=> CPU#            
        #                 / _-----=> irqs-off        
        #                | / _----=> need-resched    
        #                || / _---=> hardirq/softirq 
        #                ||| / _--=> preempt-depth   
        #                |||| /                      
        #                |||||     delay             
        #  cmd     pid   ||||| time  |   caller      
        #     \   /      |||||   \   |   /           
    	bash-3330    0.... 3531221us : sys_open (syscall_call)
    	bash-3330    0.... 3531222us : do_sys_open (sys_open)
    	bash-3330    0.... 3531222us : getname (do_sys_open)
    	bash-3330    0.... 3531223us : kmem_cache_alloc (getname)
```

Each line in the two output formats shows one function call, with one level of backtrace, along with the process name, process id, and which CPU the call was made on. The `latency_trace` output also provides information on whether interrupts have been disabled, whether a reschedule has been called for, whether its running in an interrupt context, and whether preemption has been disabled. The timestamp for the `latency_trace` output is relative to the start of the trace in microseconds; the space after the time, and before the colon, is a field that gets set to either '!' or '+' to call attention to especially long delays (in the example output it is always blank). Unfortunately, the header is a little misleading for where the "delay" and "caller" fields point. 

The sysctl `kernel.ftrace_enabled` governs whether function entry is recorded as part of the trace. It can be turned on by using either of the following commands: 

```
# sysctl kernel.ftrace_enabled=1 
        or 
        # echo 1 >/proc/sys/kernel/ftrace_enabled
```

Without that, many of the tracers are essentially pointless. 

Half-a-dozen different tracers are described in the voluminous [`Documentation/ftrace.txt`](<https://lwn.net/Articles/322731/>) that comes in the kernel source tree. In addition to the function tracer, there is the `sched_switch` tracer that tracks and reports on context switches in the kernel, showing process wakeups along with when they get scheduled. Each trace entry has a timestamp along with the priorities and states of the affected processes. 

The `nop` tracer is not a tracer at all, but can be used to clear whatever tracer is currently active. There are another seven tracers in the mainline that have yet to make it into the documentation. In addition, there are even more tracers that are targeted for the 2.6.30 kernel. 

There are four tracers that look for the maximum time spent in a particular state, recording that maximum time (in `trace_max_latency`) along with a trace of the functions called during that state. `irqsoff` looks for the longest time spent with interrupts disabled, while `preemptoff` looks for the maximum time spent with preemption turned off. Combining the two gives the `preemptirqsoff` tracer. The final tracer, `wakeup` looks for the maximum latency for the highest priority process to get scheduled after it has been woken up. 

Each of those helps by reducing the amount of trace data that a kernel hacker needs to wade through. For kernels that are configured with `CONFIG_DYNAMIC_FTRACE`, there is another way to filter the trace data. Dynamic ftrace allows the user to select kernel functions that are either included or excluded from the tracing information collected. The `available_filter_functions` file lists the functions that can be filtered, while writing function names to `set_ftrace_filter` or `set_ftrace_nofilter` will include or exclude those functions. Those lists can be appended to by using the standard `>>` shell redirection operator. In addition, there is support for wildcards so that: 

```
echo 'hrtimer_*' >/debug/tracing/set_ftrace_filter
```

will only gather tracing information from high-resolution timer functions. 

Other tracers available in the mainline kernel (i.e. what will become 2.6.29) include: 

  * `power` \- traces the CPU power state transitions 

  * `function_graph` \- similar to the function tracer, but traces both function entry and exit, which allows a call graph to be generated 

  * `sysprof` \- periodically (governed by `sysprof_sample_period`) generate stack traces for the running process or kernel thread 

  * `initcall` \- traces the entry and exit of initialization function calls during system boot 

  * `branch` \- traces branch prediction and execution 

  * `hw-branch-tracer` \- uses the Branch Target Stack (BTS) feature of x86 CPUs to trace branch execution 

  * `mmiotrace` \- traces memory-mapped I/O 

The production of human-readable output directly from the kernel recently led to some hard [questions](<https://lwn.net/Articles/322780/>) from Andrew Morton: ""Why on earth do we keep on putting all these pretty-printers and pretty-parsers into the kernel? I mean, how hard is it for userspace to read a text file, do some basic substitutions and print it out again?"" But, others argue that is precisely because of the ease of getting a readable trace directly from the kernel which makes ftrace so useful. 

Morton's [argument](<https://lwn.net/Articles/322782/>) is that the fixed format is more difficult for scripts to parse and that the output is English-only. He contends that some kind of API would simplify things. Pointing to the `latency_trace` output as an example, he said: 

[...] now you need to think about how this interface would have been designed if we'd decided to access it with something smarter than `cat'. 

I mean, look at it. All the multi-space column lining upping, the unnecessary "us" annotation, the strange symbol(symbol) thing, etc. Plus it would have been more self-describing. Right now, your parser has to either assume that the second character of "0d..1" is "irqs-off", or it has to learn how to follow ascii art lines. 

Rather than needing some kind of user-space tool to interpret the ftrace output, a kernel hacker can quickly get what they need from the kernel itself. As Ingo Molnar [pointed out](<https://lwn.net/Articles/322781/>), there may be no build environment available on the target machine, so having the trace output available directly is useful: ""Self-sufficient kernel instrumentation is a key concept to usability"". Further, he said, the output has been designed to be scriptable, but if that is not sufficient, there are options to produce raw output. Overall, he sees kernel pretty-printing as a needed feature: 

IMO pretty-printing in the kernel should not be made a religious argument but a technical argument. Pretty-printing is a tool, and as a tool it sometimes can be used for silly purposes, and sometimes it can be used for helpful purposes. You seem to argue that doing it in the kernel is always silly - and i strongly disagree with that and consider it an extreme-end position - for the reasons outlined above. 

The options that Molnar refers to are accessed via the `trace_options` file, which lists the current options settings when read, or it can be written to change options. Three separate options: raw, hex, and bin (along with several to determine which fields are included) control the format of the output. These can produce output in formats that may be easier for some tools to use. Not requiring _any_ tool to get a useful trace, though, is seen as a very important part of ftrace, at least by some. 

There are lots more options and features of ftrace than are covered here. The aforementioned `ftrace.txt` document is a good place to start for those interested in more details. 

Beyond those tracers already in the mainline, there are another handful that are being readied for 2.6.30. This includes a new event tracer, which allow tracing various scheduling, locking, and interrupt handling events. Continuous statistics gathering for workqueues and branch prediction is included as well. 

Ftrace is a useful tool that can provide excellent diagnostic information from within a running kernel. It is not a general-purpose tool, nor is it geared to people unfamiliar with the innards of the kernel, but it certainly has its uses. There is quite a bit of activity going on with ftrace these days, with numerous patches and enhancements floating around on linux-kernel. While it may not reduce Dtrace envy directly, it is a tool that many kernel hackers are using today.
