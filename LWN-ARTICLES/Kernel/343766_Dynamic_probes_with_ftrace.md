---
title: Dynamic probes with ftrace
url: https://lwn.net/Articles/343766/
date: "July 28, 2009"
category: "Development tools-Kernel tracing; Ftrace; Kprobes; Tracing-Ftrace"
author: "By Jonathan Corbet July 28, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 28, 2009

The [ftrace](<http://lwn.net/Articles/322666/>) tracing infrastructure has only been in the mainline since 2.6.27 - less than one year. During that time, ftrace has seen a great deal of development and has acquired several new capabilities. It now provides many of the features that come with more heavyweight tools like SystemTap, along with some which are unique to ftrace. But there are still capabilities found in "real" tracing utilities which are not present in ftrace. One of the more significant limitations is the lack of dynamic tracing; ftrace can easily trace function calls or use static tracepoints placed in the kernel source, but it cannot add its own tracepoints on the fly. That could change, though, should Masami Hiramatsu's [kprobe-based event tracer patch](<http://lwn.net/Articles/343345/>) make it into the mainline. 

The kprobes mechanism has been a part of the kernel for a long time; LWN ran [an overview of it](<http://lwn.net/Articles/132196/>) back in 2005\. Kprobes are, of course, dynamic tracepoints; by use of on-the-fly code patching, the kernel can hook into its own code at any point. Tools like SystemTap use kprobes to implement their dynamic tracing features. With SystemTap, though, these probes are inserted by way of a special kernel module generated on the fly - a bit of a tricky interface. Masami's patch aims to turn the insertion of dynamic probes into something close to a command-line operation. 

The patch creates a new debugfs file `/sys/kernel/debug/tracing/kprobe_events`. A new probe is inserted by appending a line to that file; that line has a somewhat complex format: 

```
p[:EVENT] SYMBOL[+offset|-offset]|MEMADDR [FETCHARGS]
        r[:EVENT] SYMBOL[+0] [FETCHARGS]
```

The first variant will set a probe with the optional name `EVENT` (if the name isn't supplied, the code makes one up). The probe will be placed at the location of the given `SYMBOL`, adjusted by the optional `offset`; an absolute address (`MEMADDR`) can also be used to locate the probe. The `FETCHARGS` portion of the line describes the data to be fetched and emitted when the tracepoint is hit; the syntax allows the specification of various types of data, including register contents, stack offsets, absolute addresses, kernel symbols, function arguments, and more. What the code does not currently allow is much in the way of sophisticated formatting of this data; it comes out in straight hexadecimal format. 

The second line, above, inserts a "retprobe" instead. Retprobes are fired when the given function (as specified by `SYMBOL`) returns to its caller; they can capture the function's return value and the address it is returning to. 

The patch posting contains an example of a couple of probes placed in `do_sys_open()`; the commands to do so are: 

```
echo p:myprobe do_sys_open a0 a1 a2 a3 > /sys/kernel/debug/tracing/kprobe_events
        echo r:myretprobe do_sys_open rv ra >> /sys/kernel/debug/tracing/kprobe_events
```

Two probes are placed here. One called `myprobe` will fire on entry to `do_sys_open()` and output the values of the four arguments passed to that function. The other, `myretprobe`, triggers when `do_sys_open()` returns, fetching the return value and return address in the process. 

The output from these tracepoints can be seen by reading `/sys/kernel/debug/tracing/trace`: 

```
#           TASK-PID    CPU#    TIMESTAMP  FUNCTION
    #              | |       |          |         |
               <...>-1447  [001] 1038282.286885: do_sys_open+0x0/0xd6: 0xffffff9c 0x40413c 0x8000 0x1b6
               <...>-1447  [001] 1038282.286915: sys_open+0x1b/0x1d <- do_sys_open: 0x3 0xffffffff81367a3a
```

Here we see a call to `do_sys_open()` with its four parameters: the directory file descriptor (`0xffffff9c`), file name pointer (`0x40413c`), flags (`0x8000`), and mode (`0x1b6`). For the curious, the strange file descriptor value is the magic value `AT_FDCWD`, meaning that the file lookup should begin in the current working directory. There is also a return line (as indicated by the "`<-`" arrow) showing that the call returned to `sys_open()`, having opened file descriptor 3\. 

The patch also provides mechanisms for turning individual probes on and off, filtering probe output, and maintaining profiles of probe hits. 

Tracing of function entry and exit as shown above is a useful feature, but the existing ftrace function tracer can do that already. The obvious value in this new patch is the ability to place tracepoints at locations other than function entry and exit points. But that leads to an interesting question: how does the user manage to get tracepoints set in the right locations? Guessing at offsets from function symbols seems like a recipe for trouble, especially given that the placement of a tracepoint in the middle of an instruction is unlikely to lead to pleasant results. 

Addressing that last concern is, as it turns out, the job of the bulk of the code in Masami's patch. Placing probes is relatively easy - the code to do that is already in the kernel. But making sure that the probe is in the right place requires the addition of an x86 instruction decoding module. When a probe is requested within a function, the instruction decoder goes to work; it starts at the beginning of the function and decodes instructions until it reaches the probe point. If the probe is located at an instruction boundary, all is well; otherwise the placement of the probe is disallowed. 

Actually generating the right offsets for dynamic probes is likely to be a job for user-space software which can parse debugging information and map line numbers onto offsets. A tool like a debugger or SystemTap, for example. It is, in fact, conceivable that tools like SystemTap could move over to this mechanism once it's merged; that would allow SystemTap to share more of the low-level ftrace plumbing and get it closer to working with unpatched mainline kernels. 

That's getting a little ahead of the game, though; first the kprobe-based event tracing code needs to be merged. There does not appear to be any real opposition to that merger - but this code has been around for a while and is currently on its 13th revision. The value of getting real dynamic probing support into the kernel seems reasonably evident, though; expect this feature to get in at some point.
