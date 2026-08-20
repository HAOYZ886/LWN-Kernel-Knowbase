---
title: Another mainline push for utrace
url: https://lwn.net/Articles/364584/
date: "December 2, 2009"
category: Utrace; ptrace
author: "By Jake Edge December 2, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
December 2, 2009

When last we [looked](<http://lwn.net/Articles/325180/>) in on utrace, back in March, it was being proposed for inclusion into 2.6.30. There were various objections at that time, but the biggest was the lack of a "real" in-kernel user for utrace. It was suggested that providing a real user along with utrace itself would smooth its path into the mainline. Now utrace has [returned](<https://lwn.net/Articles/364705/>) in the form of a set of patches from Oleg Nesterov (based on Roland McGrath's work), along with a rewrite of the `ptrace()` system call using the utrace interface. With the 2.6.33 merge window opening soon, the hope is that utrace will, finally, make its way into the mainline. 

Utrace provides a means to control user-space threads, which could be used for debugging, tracing, and other tasks like user-mode-linux. SystemTap is one of the biggest current utrace users, as Red Hat and Fedora kernels have had utrace support for several years. Utrace came from a recognition that `ptrace()` was too limited--and messy--for many of the things folks wanted to use it for. In particular, only allowing one active tracing process for a given thread, as `ptrace()` requires, was too limiting for various envisioned tracing and control scenarios. Utrace allows multiple tracing "engines" to attach to a thread, list which events they are interested in, and receive callbacks when those events occur. 

The interface provided by utrace has not changed enormously since our first [look](<http://lwn.net/Articles/224772/>) in March 2007. Engines, which are typically implemented as loadable kernel modules, will attach to a given thread by using `utrace_attach_task()` or `utrace_attach_pid()` depending on whether they have a `struct task_struct` or `struct pid` available. In either case, a `struct utrace_engine` pointer is returned, which is used to identify the engine in additional calls. 

The `struct utrace_engine` looks like: 

```
struct utrace_engine {
            const struct utrace_engine_ops *ops;
            void *data;
            unsigned long flags;
        };
```

with `flags` containing an event mask and `data` used for engine-specific private data. The most interesting part is the `ops` field which points to a set of ten different callback functions. These functions make up the heart of the tracing engine functionality. 

The function pointers in `struct utrace_engine_ops` are [described](<http://people.redhat.com/roland/utrace/DocBook/re10.html>) in `linux/utrace.h`. All of the kerneldoc comments are pulled from the source code files into the [DocBook documentation](<http://people.redhat.com/roland/utrace/DocBook/>) that comes with the patchset. The callbacks are made as the traced thread encounters various events. These include signals being delivered, `clone()` or `exec()` being called, other system calls as they are entered or exited, thread exit or death, and more. In each case, the callbacks are made for each interested engine in the order in which the engines were attached. 

An engine uses the `utrace_set_events()` (or `utrace_set_events_pid()`) call to indicate which of the events it is interested in: 

```
int utrace_set_events(struct task_struct *target,
                              struct utrace_engine *engine,
                              unsigned long events);
```

The `UTRACE_EVENT()` macro is used to turn on the appropriate bits in the `events` mask. There must be a callback defined in the `engine->ops` table for any events which are enabled. 

Once a callback has been invoked, the engine uses `utrace_control()` (or `utrace_control_pid()`) to tell the traced thread to do something: 

```
int utrace_control(struct task_struct *target,
                           struct utrace_engine *engine,
                           enum utrace_resume_action action);
```

The `action` parameter governs what is supposed to happen. Those actions include things like single-stepping, block-stepping, resuming execution, detaching from the thread, and so on. 

In the only real complaint about the patchset seen so far, Christoph Hellwig is [unhappy](<https://lwn.net/Articles/364769/>) that the `ptrace()` reimplementation is not supplanting the current `ptrace()` code: ""One thing I really hate about this is that it introduces two ptrace implementations by adding the new one without removing the old one."" In the patches, the inclusion of utrace is governed by the `CONFIG_UTRACE` flag. Since it isn't optional to have a `ptrace()` system call, that meant the current code needed to stay. 

What Hellwig suggests, though, is adding utrace support to the two major architectures that don't have it (arm and mips), then removing the current `ptrace()`. He clearly believes it is too late to get utrace into 2.6.33, which would allow time to get utrace support into those--and hopefully other, minor architectures--before utrace is merged. ""If the remaining minor architectures don't manage to get their homework done they're left without ptrace,"" he said. 

That didn't sit well with various other kernel hackers. Pavel Machek [said](<https://lwn.net/Articles/364772/>): ""I don't think introducing regressions to force people to rewrite code is a good way to go"". In addition, Ingo Molnar seems to have warmed up to utrace's inclusion since it was last proposed. Molnar had many complaints about utrace last time, but is much more positive this time. He doesn't [think](<https://lwn.net/Articles/364774/>) adding more architecture support is the way to go: 

Regarding porting it to even more architectures - that's pretty much the worst idea possible. It increases maintenance and testing overhead by exploding the test matrix, while giving little to [the] end result. Plus the worst effect of it is that it becomes even more intrusive and even harder (and riskier) to merge. 

Unlike last time, where most of the complaints were not aimed at the code itself, but more at its timing and lack of an in-kernel user, this time there is some code review taking place. Peter Zijlstra has a fairly detailed [review](<https://lwn.net/Articles/364779/>) of both the code and documentation for example. There is a clear sense that utrace is clearing hurdles that may have held it up in the past. 

One of the outcomes from the tracing meetings at the Collaboration Summit in April was to come up with an in-kernel user, and `ptrace()` seemed like a good candidate. Other ideas were mentioned in those meetings, including adding a gdb "stub" in the kernel to allow debugging of user-space programs. A [patch](<http://lwn.net/Articles/364268/>) to expose a `/proc/PID/gdb` interface that implements gdb's remote serial protocol was proposed by Srikar Dronamraju. 

That patch is running into more serious difficulty than utrace seems to be. Because `kgdb` already exposes the remote serial interface for gdb, but for the kernel instead, Zijlstra and Molnar think that the two should be combined. It seems unlikely to get merged until that is resolved. 

Some parts of the utrace patchset have spent time in the -mm tree, and utrace has been shipped with every Fedora kernel since FC6. But the utrace-ptrace piece has not done any time in either -mm or -next, which may make it harder to get it in the mainline for 2.6.33. Since utrace is optional, though, there are relatively few risks. McGrath is willing to [consider](<https://lwn.net/Articles/364782/>) removing the current `ptrace()` implementation, but its clear that he and Nesterov--maintainers of the current `ptrace()`--would prefer to get utrace into the mainline now: 

We don't want to rush anyone, like uninterested arch maintainers. We don't want to break anyone who doesn't care about our work (we do test for ptrace regressions but of course new code will always have new bugs so some instances of instability in the transition to a new ptrace implementation have to be expected no matter how hard we try). We just don't want to keep working out of tree. 

Presumably, we will know within the next few weeks whether utrace makes its way into 2.6.33. But, if that doesn't happen, it would seem that one more kernel development cycle is all that it should take.
