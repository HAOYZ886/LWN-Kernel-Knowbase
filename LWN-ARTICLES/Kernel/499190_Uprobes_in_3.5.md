---
title: Uprobes in 3.5
url: https://lwn.net/Articles/499190/
date: "May 30, 2012"
category: Tracing; Uprobes
author: "By Jonathan Corbet May 30, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 30, 2012

Uprobes is a kernel patch with a long story and many contentious discussions behind it. This code has its roots in utrace, a user-space tracing and debugging API that was first [covered here](<https://lwn.net/Articles/224772/>) in early 2007. Utrace ran into various types of opposition (only partly related to its own origin in SystemTap) and has never been merged, but a piece of it lives on in the form of uprobes, which is charged with the placement of probes into user-space code. After several mailing-list rounds of its own, uprobes was finally merged for the 3.5 kernel development cycle. Just how this facility will be used remains to be seen, however. 

At the core of uprobes is this function: 

```
#include <linux/uprobes.h>
    
        int uprobe_register(struct inode *inode, loff_t offset, struct uprobe_consumer *uc);
```

The `inode` structure represents an executable file; the probe is to be placed at `offset` bytes from the beginning. The `uprobe_consumer` structure tells the kernel what is to be done when a process encounters the probe; it looks like: 

```
struct uprobe_consumer {
    	int (*handler) (struct uprobe_consumer *self, struct pt_regs *regs);
    	bool (*filter) (struct uprobe_consumer *self, struct task_struct *task);
    	struct uprobe_consumer *next;
        };
```

The `filter()` function is optional; if it exists, it determines whether `handler()` is called for each specific hit on the probe. The handler returns an `int`, but the return value is ignored in the current code. 

Since probes are associated with files, they affect all processes that run code from those files. A special copy is made of the page to contain the probe; in that copy, the instruction at the specified offset is copied and replaced by a breakpoint. When the breakpoint is hit by a running process, `filter()` will be called if present, and `handler()` will be run unless the filter said otherwise. Then the displaced instruction is executed (using the "execute out of line" mechanism described in [this article](<https://lwn.net/Articles/370322/>)) and control returns to the instruction following the breakpoint. 

Uprobes thus implements a mechanism by which a kernel function can be invoked whenever a process executes a specific instruction location. One could imagine a number of things that said kernel function could do; there has been talk, for example, of using uprobes (and, perhaps someday, something derived from utrace) as a replacement for the much-maligned `ptrace()` system call. Tools like GDB could place breakpoints with uprobes; it might even be possible to load simple filters for conditional breakpoints into the kernel, speeding their execution considerably. Uprobes could also someday be a component of a Dtrace-like dynamic tracing functionality. For now, though, the interfaces for that kind of feature have not been added to the kernel; none have even been proposed. 

What the current implementation _does_ have is integration with the perf events subsystem. New dynamic "events" can be added to any file location via an interface similar to that used for [dynamic kernel tracepoints](<https://lwn.net/Articles/343766/>). In particular, there is a new file called `uprobe_events` in the tracing directory (`/sys/kernel/debug/tracing/` on most systems) that is used to add and remove events. As an example, a line like: 

```
echo 'p:bashme /bin/bash:0x4245c0' > /sys/kernel/debug/tracing/uprobe_events
```

would place a new event (called "`bashme`") at location 0x4245c0 in the `bash` executable. The event would then appear with all other events in `/sys/kernel/debug/tracing/events`, in the `uprobes` subdirectory. Like other events, it is not actually turned on until its `enabled` attribute is set. See [Documentation/trace/uprobetracer.txt](<https://lwn.net/Articles/499286/>) for details on the interface at this level. 

Placing uprobes is, by default, a privileged operation requiring the `CAP_SYS_ADMIN` capability. One can remove the privilege requirement by setting the `perf_paranoid` sysctl knob to `-1`, but doing so will allow the placement of dynamic tracepoints anywhere in the system, in kernel or user space. Thus, one need not be overly paranoid to leave `perf_paranoid` at its default setting. 

The `perf` tool has been enhanced to make working with dynamic user-space tracepoints easy. One can, for example, set a tracepoint at the entry to the C library's `malloc()` implementation with: 

```
perf probe -x /lib64/libc.so.6 malloc
```

That tracepoint can then be treated like any other event understood by perf. See [the explanatory text](<http://git.kernel.org/linus/654443e20dfc0617231f28a07c96a979ee1a0239>) from Ingo Molnar's pull request for examples of what can be done. 

Most kernel patches are conceived, implemented, reviewed, and merged into the mainline over a fairly short period of time. But some of them seem to languish for years without making much progress. Uprobes was such a patch set. It must have been frustrating for the developers to keep revising and posting this code, only to see it shot down over and over again. But the kernel community can be supportive of developers who show both persistence and a willingness to listen to criticism. The result, in this case, is a user-space probing mechanism that has been simplified, made more robust, and integrated into the existing events infrastructure. Hopefully it was worth the wait.
