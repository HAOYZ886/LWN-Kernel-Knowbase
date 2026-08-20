---
title: A SystemTap update
url: https://lwn.net/Articles/315022/
date: "January 21, 2009"
category: SystemTap
author: "January 21, 2009 This article was contributed by Mark Wielaard"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

January 21, 2009

This article was contributed by Mark Wielaard

[SystemTap](<http://sourceware.org/systemtap/>) has been under active development for a some years. More than 35 people have [contributed enhancements](<http://sourceware.org/systemtap/wiki/Myths>) in the last year. But newer developments, like the ability to dynamically trace user space programs, seem to have been very quietly introduced and, thus, have not always been noticed by users that are not yet using SystemTap extensively. So this article will take a look at what currently works out of the box, what that box should contain to make things work, the work in progress, and the challenges SystemTap faces to be more powerful and get more widespread adoption. 

SystemTap's goal is to provide full system observability on production systems, which is safe, non-intrusive, (near) zero-overhead and which allows ubiquitous data collection across the whole system for any interesting event that could happen. To achieve this goal, SystemTap defines the [stap language](<http://sourceware.org/systemtap/langref/langref.html>), in which the user defines probes, actions, and data acquisition. The SystemTap translator and runtime guarantees that probe points are only placed on safe locations and that probe functions cannot generate too much overhead when collecting data. For dynamic probes on addresses inside the kernel, SystemTap uses [kprobes](<http://lwn.net/Articles/132196/>); for dynamic probes in user space programs, instead, SystemTap uses its cousin [uprobes [PDF]](<http://ols2006.108.redhat.com/2007/Reprints/keniston-Reprint.pdf>). This provides a unified way of probing and then collecting data for observing the whole system. To dynamically find locations for probe points, arguments of the probed functions and the variables in scope at the probe point, SystemTap uses the debuginfo ([Dwarf](<http://dwarfstd.org/>)) standard debugging information that the compiler generates. 

So, to provide an ideal setting for using SystemTap, GNU/Linux distributions should provide easy access to debuginfo for the kernel and user space programs. Almost all distributors do this. The kernel supports kprobes, which has been in the upstream kernel for some years, and uprobes, which comes with (and is automatically loaded by) SystemTap, but which relies on the full utrace framework, which isn't yet in the mainline kernel. (The latest few releases of the Fedora family, including Red Hat Enterprise Linux and CentOS, do include full utrace support by default). SystemTap works without debuginfo, but the range of probes and the amount of data you can collect is then very limited. And it works without utrace support, but then you won't be able to do deep user space probing, only observe direct user/kernel space interactions. 

There are various probe variants one can use with SystemTap, but the most interesting ones are the debuginfo-based probes for the kernel, kernel modules, and user space applications. These can use function, statement or return variants, and wildcards, such as: 

  * `kernel.function("rpc_new_task")`: a named kernel function, 
  * `process("/bin/ls").function("*")`: any function entry in a specific process, 
  * `module("usb*").function("*sync*").return`: every return of a function containing the word `sync`, in any module starting with `usb`, or 
  * `kernel.statement("bio_init@fs/bio.c+3")`: for a specific statement in a particular file 

Depending on the type of probe, one can access specifics of the probe point. For the debuginfo based probes these are `$var` for in-scope variables or function arguments, `$var->field` for accessing structure fields, `$var[N]` for array elements, `$return` for the return value of a function in a return probe, and meta variables like `$$vars` to get a string representation of all the in-scope variables at a particular probe point. All access to such constructs are safeguarded by the SystemTap runtime to make sure no illegal accesses can occur. 

Given that one has the debuginfo of a program installed, one can easily get a simple call trace of a specific program, including all function parameters and return values with the following stap script: 

```
probe process("/bin/ls").function("*").call
      {
        printf("=>%s(%s)\n", probefunc(), $$parms);
      }
    
      probe process("/bin/ls").function("*").return
      {
        printf("<=%s:%s\n", probefunc(), $$return);
      }
```

The [examples included with SystemTap](<http://sourceware.org/systemtap/examples/keyword-index.html#CALLGRAPH>) come with much more powerful versions that show timed, per-thread call graphs, optionally showing only children of a particular function call. 

While these probing and data extraction constructs are powerful, they do require some knowledge of the kernel or program code base. Since you are often interested in _what_ is happening and not precisely _how_ , SystemTap comes with "tapsets," which are utility functions and aliases for groups of interesting probes in a particular subsystem. Examples include system calls, NFS operations, signals, sockets, etc. Currently these tapsets are distributed with SystemTap itself, but ideally each program or subsystem would come with its own tapset of interesting events provided by the program or subsystem maintainer. 

Just printing out events while they occur is not always ideal. First, you may be overwhelmed by volume of the output; second, you might only be interested in a specific subset of the same event (only certain parameters, only calls that take longer than a specific time, only from the process that does the most calls over a specific time frame, etc.). Finally, processing all the events on your production system might interfere with the thing you are trying to observe. Especially at the start of your investigations, when you might not yet be sure what the interesting events are, you may do some very wide probing to see what is going on. 

For this reason the stap language supports variables that can be used as [associative arrays](<http://sourceware.org/systemtap/langref/node9.html>), simple [control structures](<http://sourceware.org/systemtap/langref/node8.html>) and [data aggregation functions](<http://sourceware.org/systemtap/langref/node10.html>) to do simple statistics during probe time, with very low overhead and without having to call external programs that might interfere with the system being probed. 

The following script might be how you would start investigating a problem involving a system which seems to do an excessive amount of reads. It uses the "vfs" tapset and an associative array to store the number of reads a particular executable with a specific process ID does: 

```
global totals;
      probe vfs.read
      {
        totals[execname(), pid()]++
      }
    
      probe end
      {
        printf("== totals ==\n")
        foreach ([name,pid] in totals-)
          printf("%s (%d): %d \n", name, pid, totals[name,pid])
      }
```

This will give you a list of executables and their pid sorted by the total number of vfs reads done while the script was running. These facilities in the stap language help greatly to minimize any overhead of the tracing framework. If you would try to do the same thing by just printing each vfs event and then post-processing the results with Perl, you might end up with Perl itself being the process doing the most vfs calls, or worse, by having to parse megabytes of trace data, Perl might start trashing the system even more, making it harder to determine the root cause of the original problem. 

SystemTap now also supports [static markers](<http://lwn.net/Articles/245671/>) in the kernel. This allows subsystem maintainers to mark specific events as interesting, providing a format string of the arguments to the event that can be easily parsed by tracing tools. The advantage of static markers over tapsets is that they are in-code and so might be easier to maintain, though you probably still want to have an associated tapset for utilities to nicely format the arguments or associate various markers with each other. Also, they can work without needing any DWARF debuginfo around, but you lose the ability to inspect local variables or function parameters not passed to the marker. You use them with a command like: 

```
probe kernel.mark("kernel_sched_wakeup")
```

The tapset can then access the arguments through `$argN` and get the argument format string of the marker with `$format`. 

An alternate way of adding static markers to the kernel, [tracepoints](<http://lwn.net/Articles/291091/>), is not yet directly supported in SystemTap. Tracepoints have the disadvantage that they require the DWARF debuginfo to be around because they don't currently specify the types of their arguments except through their function prototypes. So SystemTap can currently only use tracepoints via hand-written intermediary code that maps them to markers. 

The development version of SystemTap recently got support for user-space static markers. Although SystemTap defines its own STAP_PROBE macros for usage in applications that want to add static markers, there is also an alternative tracing tool, [Dtrace](<http://lwn.net/Articles/244536/>), that has its own way for programs to embed static markers. SystemTap supports the convention used by Dtrace by providing an alternative include file and build preprocessor so that programs using DTRACE_PROBE macros can be compiled as if for Dtrace and have their static markers show up with SystemTap. 

Luckily, there are various programs that already have such markers defined. For example [PostgreSQL](<http://www.postgresql.org/docs/8.2/static/dynamic-trace.html>) has various static markers to trace higher-level events like transactions and database locks. Currently one has to [adapt the build process](<http://sourceware.org/systemtap/wiki/UsingStaticUserMarkers>) of such programs by hand, but the next version of SystemTap will come with scripts that will automate that process.

While SystemTap works well on GNU/Linux distributions that support it, there are a couple of challenges to overcome to make it more ubiquitous and easier for more people to use out of the box. This goes beyond work on the SystemTap code base itself. Since the goal is to provide full system observability, from low-level kernel events to high-level application events, there is work to be done all across the GNU/Linux stack. Also needed is better integration into more distributions, providing default installation of SystemTap and tapsets, easy access to debuginfo for deep inspection, binaries compiled with marker support for high-level events, etc. The two main challenges to make SystemTap more powerful and easier to use on any distribution are debuginfo and better kernel support. 

A lot of power of SystemTap comes from the fact that it can use DWARF debuginfo from the kernel and applications to do very detailed inspection. But this power comes at a price, since the debuginfo is often large. For example, on Fedora, the kernel debuginfo package is far larger than the kernel package itself. One easy win will be to split the debuginfo package into the DWARF files and the source files, which are needed for a debugger, but not directly for a tracer like SystemTap. Fedora [plans to do this](<https://fedoraproject.org/wiki/Features/DebugInfoRevamp>) for its next release. The elfutils team is also working on [a framework for Dwarf transformation and compression](<https://fedorahosted.org/elfutils/wiki/DwarfTasks>) that could be used as post-processor on the output of the compiler. 

SystemTap sometimes suffers from the same issues you might have with a debugger: the compiler has optimized the code, but forgot where it put a certain variable after the optimization. Of course this is always the variable you are most interested in. Alexandre Oliva is working on improving the local variable debug information in GCC. His [variable tracking assignments](<http://www.lsd.ic.unicamp.br/~oliva/papers/vta/paper.pdf>) [PDF] branch in GCC aims to improve debug information by annotating assignments early in the compilation process and carrying over such annotations throughout all optimization passes so that you can always accurately track variables, even in optimized code. 

Finally, there is work being done on having a SystemTap "client and server" that could be used on production systems where you might not even want to have any tools or debuginfo installed. You can then set up a development client that has the same configuration as the production system with the addition of the SystemTap translator and all debuginfo, create and test your scripts there. The final result of this work could then be used on the bare-bones production server.

Most of the SystemTap runtime, like the kprobes support, is maintained in the upstream linux kernel, but there is some stuff still missing. This leads to distributions having to add patches to their kernel, especially to support user space tracing. In particular, the utrace framework is still not upstream. Over the last few kernel releases, various parts have been merged, including the utrace [user_regset](<http://lwn.net/Articles/263204/>) framework, which creates an interface for code accessing the user-space view of any machine specific state, and the [tracehook](<http://lwn.net/Articles/290954/>) work, which provides a framework for all the user process tracing. The actual utrace framework sits on top of these components; the `ptrace()` interface is implemented as utrace client. Anything that changes the ptrace implementation is hairy stuff, so there is a large [ptrace testsuite](<http://sourceware.org/systemtap/wiki/utrace/tests>) to make sure that nothing breaks. One idea under consideration is to [push utrace upstream in two installments](<https://www.redhat.com/archives/utrace-devel/2009-January/msg00039.html>). At first, using utrace or ptrace on a process would be [mutually exclusive](<https://www.redhat.com/archives/utrace-devel/2009-January/msg00040.html>). That could pave to path to get pure-utrace upstream in first and then do proper ptrace cooperation in a second go.

This approach would also provide the way for uprobes, which depends on the utrace framework, to be submitted upstream. Uprobes components such as breakpoint insertion and removal and the single-stepping infrastructure are also potentially useful for other user space tracers and debuggers. Like with utrace, one idea is to factor out these portions of uprobes so that it can be used by multiple clients as a shared [user-space breakpoint support](<https://www.redhat.com/archives/utrace-devel/2008-November/msg00020.html>) (ubs) layer. With multiple clients using the same layer, upstream acceptance might be easier.

One candidate for using both the utrace and the uprobes layer besides SystemTap is [Froggy](<http://sourceware.org/cgi-bin/cvsweb.cgi/froggy/?cvsroot=systemtap>), which provides an alternative debugger interface to ptrace. The [GDB Archer](<http://sourceware.org/gdb/wiki/ProjectArcher>) project would like to serve as testbed for Froggy, which they hope will also make GDB more robust when linked with libpython, which is being used for GDB scripting.

In the past, kernel maintainers were skeptical about tracing, which resulted in tracing frameworks like dprobes, LTT and parts of the SystemTap runtime being maintained outside the main kernel tree. But now that there is actually [no shortage of tracing options](<http://lwn.net/Articles/291091/>) in the kernel, people like Ted Ts'o have been urging the SystemTap hackers to push as much as possible upstream. Ted also encourages the developers to focus more on the kernel hackers as first-rate customers, rather than focusing exclusively on the whole system experience for production setups. The SystemTap developers have been successful in making their module support "just work" with any kernel. It currently works with kernel versions between 2.6.9 and the latest, 2.6.28; it is also regularly tested against the latest -rc kernels. But, maybe they have been a little too successful, because having this activity be more visible on the linux kernel mailing list would be good publicity. In response, there is now an active SystemTap bug called "[Make upstream kernel developers happy](<http://sourceware.org/bugzilla/show_bug.cgi?id=7042>)" that calls for more frequent postings on the main kernel mailing list, improvements in the usage of debuginfo as described above, and pushing utrace and uprobes upstream first as a priority. 

There is still work to do, but over the last couple of years the GNU/Linux tracing and debugging experience has kept improving. Hopefully soon, all these parts will fall into place and provide hackers with a fairly nice environment for not only debugging on development systems, but also for unobtrusive tracing on production systems. 

About the author: Mark Wielaard is a Senior Software engineer at Red Hat working in the Engineering Tools group hacking on SystemTap.
