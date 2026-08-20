---
title: Ktap — yet another kernel tracer
url: https://lwn.net/Articles/551314/
date: "May 22, 2013"
category: "Development tools-Kernel tracing; Tracing"
author: "By Jonathan Corbet May 22, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 22, 2013

Once upon a time, usable tracing tools for Linux were few and far between. Now, instead, there is a wealth of choices, including the in-kernel ftrace facility, SystemTap, and the LTTng suite; Oracle also has [a port of DTrace](<https://blogs.oracle.com/linux/entry/announcement_dtrace_for_oracle_linux>) for its distribution, available to its paying customers. On May 21, another alternative showed up in the form of the [ktap 0.1 release](<https://lwn.net/Articles/551253/>). Ktap does not offer any major features that are not available from the other tracing tools, but there may still be a place for it in the tracing ecosystem. 

Ktap appears to be strongly oriented toward the needs of embedded users; that has affected a number of the design decisions that have been made. At the top of the list was the decision to embed a byte-code interpreter into the kernel and compile tracing scripts for that interpreter. That is a big difference from SystemTap, which, in its current implementation, compiles a tracing script into a separate module that must be loaded into the kernel. This difference matters because an embedded target often will not have a full compiler toolchain installed on it; even if the tools are available, compiling and linking a module can be a slow process. Compiling a ktap script, instead, requires a simple utility to produce byte code for the ktap kernel module. 

That compiler implements a language that is based on [Lua](<http://www.lua.org/>). It is C-like, but it is dynamically typed, has a dictionary-like "table" type, and lacks arrays and pointers. There is a simple function definition mechanism which can be used like this: 

```
function eventfun (e) {
    	printf("%d %d\t%s\t%s", cpu(), pid(), execname(), e.tostring())
        }
```

The resulting function will, when called, output the current CPU number, process ID, executing program name, and the string representation of the passed-in event `e`. There is a probe-placement function, so ktap could arrange to call the above function on system call entry with: 

```
kdebug.probe("tp:syscalls", eventfun)
```

A quick run on your editor's system produced a bunch of output like: 

```
3 2745	Xorg	sys_setitimer(which: 0, value: 7fff05967ec0, ovalue: 0)
        3 2745	Xorg	sys_setitimer -> 0x0
        2 27467	as	sys_mmap(addr: 0, len: 81000, prot: 3, flags: 22, fd: ffffffff, off: 0)
        2 27467	as	sys_mmap -> 0x2aaaab67c000
        2 3402	gnome-shell	sys_mmap(addr: 0, len: 97b, prot: 1, flags: 2, fd: 21, off: 0)
        2 3402	gnome-shell	sys_mmap -> 0x7f4ec4bfb000
```

There are various utility functions for generating timer requests, creating histograms, and so on. So, for example, this script: 

```
hist = {}
    
        function eventfun (e) {
    	if (e.sc_is_enter) {
    	    inplace_inc(hist, e.name)
    	}
        }
    
        kdebug.probe("tp:syscalls", eventfun)
    
        kdebug.probe_end(function () {
    	histogram(hist)
        })
```

is sufficient to generate a histogram of system calls over the period of time from when it starts until when the user interrupts it. Your editor ran it with a kernel build running and got output looking like this: 

```
value ------------- Distribution ------------- count
            sys_enter_open |@@@@@@@@                               587779    
           sys_enter_close |@@@@                                   343728    
        sys_enter_newfstat |@@@@                                   331459    
            sys_enter_read |@@@                                    283217    
            sys_enter_mmap |@@@                                    243458    
           sys_enter_ioctl |@@                                     219364    
          sys_enter_munmap |@@                                     165006    
           sys_enter_write |@                                      128003    
            sys_enter_poll |@                                      77311     
        sys_enter_recvfrom |                                       52898
```

The syntax for setting probe points closely matches that used by perf; probes can be set on specific functions or tracepoints, for example. It is possible to hook into the perf events mechanism to get other types of hardware or software events, and memory breakpoints are supported. The (sparse) documentation packaged with the code also suggests that ktap is able to set user-space probes, but none of the example scripts packaged with the tool demonstrate that capability. 

Ktap scripts can manipulate the return value of probed functions within the kernel. There does not currently appear to be a way to manipulate kernel-space data directly, but that could presumably be added (along with lots of other features) in the future. What's there now is a proof of concept as much as anything; it is a quick way to get some data out of the kernel but does not offer a whole lot that is not available using the existing ftrace interface. 

For those who want to play with it, the first step is a simple: 

```
git clone https://github.com/ktap/ktap.git
```

From there, building the code and running the sample scripts is a matter of a few minutes of relatively painless work. There is the `ktapvm` module, which must, naturally, be loaded into the kernel. That module creates a special virtual file (`ktap/ktapvm` under the debugfs root) that is used by the `ktap` binary to load and run compiled scripts. 

Ktap in its current form is limited, without a lot of exciting new functionality. Even so, it seems to have generated a certain amount of interest in the development community. Getting started with most tracing tools usually seems to involve a fair amount of up-front learning; ktap, perhaps, is a more approachable solution for a number of users. The whole thing is about 10,000 lines of code; it shouldn't be hard for others to run with and extend. If developers start to take the bait, interesting things could happen with this project.
