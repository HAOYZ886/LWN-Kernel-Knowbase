---
title: The return of utrace
url: https://lwn.net/Articles/325180/
date: "March 25, 2009"
category: Utrace
author: "By Jake Edge March 25, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
March 25, 2009

An in-kernel tracing infrastructure for user-space code, utrace, has long been in a kind of pending state; it has shipped in every Fedora kernel since Fedora Core 6, and has done some time in the -mm tree, but it has never gotten into the mainline. That may now be changing, given a recent [push for inclusion](<http://lwn.net/Articles/324753/>) of the core utrace code. There are some lingering questions about including utrace, at least for 2.6.30, because the patchset doesn't add any in-kernel user of the interface. 

Utrace grew out of Roland McGrath's work on maintaining the `ptrace()` system call. That call is used by user-space programs to do things like trace system calls using `strace`, but it is also used in less obvious ways--to implement user-mode-linux (UML) for example. While `ptrace()` has generally sufficed, it is, by all accounts, a rather ugly and flawed interface both for kernel hackers to maintain and for developers to use. McGrath [described the genesis of utrace](<https://lwn.net/Articles/325247/>) in a recent linux-kernel post: 

I hatched the essential design of utrace when I'd recently spent a whole lot of time fixing the innards of ptrace and a whole lot of time helping userland implementors of debuggers and the like figure out how to work with ptrace (and hearing their complaints about it). At the same time, the group I'm in (still) was contemplating both the implementation issues of a generic debugger, how to make it tractable to work up to far smarter debuggers, and also the design of what became systemtap. 

Basically, utrace implements a framework for controlling user-space tasks. It provides an interface that can be used by various tracing "engines", implemented as loadable kernel modules, that wish to be notified of events that occur on threads of interest. As might be expected, engines register callback functions for specific events, then attach to whichever thread they wish to trace. 

The callbacks are made from "safe" places in the kernel, which allows the functions great leeway in the kinds of processing they can do. No locks are held when the callbacks are made, so they can block for a short time (in calls like `kmalloc()`), but they shouldn't block for long periods. Doing so, risks making the `SIGKILL` signal from working properly. If the callback needs to wait for I/O or block on some other long-running activity, it should stop the execution of the thread and return, then resume the thread when the operation completes. 

There are various events that can be watched via utrace: system call entry and exit, `fork()`, signals being sent to the task, etc. Single-stepping through a task being traced can also be handled via utrace. One of the benefits that utrace provides, which `ptrace()` lacks, is the ability to have multiple engines tracing the same task. Utrace is well documented in [DocBook manual](<http://people.redhat.com/roland/utrace/DocBook/>) included with the patch. 

LWN first [looked at utrace](<http://lwn.net/Articles/224772/>) just over two years ago, but, since then, it has largely disappeared from view. Reimplementing `ptrace()` using utrace is certainly one of the goals, but the current patches do not do that. But, there is a fundamental disagreement between McGrath and other kernel hackers about whether utrace can be merged without it. The problem is that there is no in-tree user of the new interface, and, as Ted Ts'o [put it](<https://lwn.net/Articles/325267/>), ""we need to have a user for the kernel interface along with the new kernel interface"". 

The proposed utrace patchset consists of a small patch to clean up some of the tracehook functionality, a large 4000 line patch that implements the utrace core, and another patch that adds an ftrace tracer that is based on utrace event handling. The latter, implemented by SystemTap developer Frank Eigler, would provide an in-tree user of the new utrace code, but received a rather [chilly response](<https://lwn.net/Articles/325259/>) from Ingo Molnar: ""[...] without the ftrace plugin the whole utrace machinery is just something that provides a _ton_ of hooks to something entirely external: SystemTap mainly."" 

Therein lies one of the main concerns expressed about utrace. The utrace-ftrace interface is not seen as a real user of utrace, more of a ""big distraction"", as Andrew Morton [called it](<https://lwn.net/Articles/325268/>). The worry is that adding utrace just makes it easier to keep SystemTap out of the mainline. While the kernel hackers have some serious reservations about the specifics of the SystemTap implementation, they would like to see it head towards the mainline. The fear is that by merging things like utrace, it may enable SystemTap to stay out of the mainline that much longer. Molnar posted his [take](<https://lwn.net/Articles/325294/>) on the issue, concluding: 

Putting utrace upstream now will just make it more convenient to have SystemTap as a separate entity - without any of the benefits. Do we want to do that? Maybe, but we could do better i think. 

In addition, Molnar is not pleased that the utrace changes haven't been reviewed by the ftrace developers and were submitted just as the merge window for 2.6.30 is about to open. He believes that McGrath, Eigler, and the other utrace developers should be working with the ftrace team: 

kernel/utrace.c should probably be introduced as kernel/trace/utrace.c not kernel/utrace.c. It also overlaps pending work in the tracing tree and cooperation would be nice and desired. 

The ftrace/utrace plugin is the only real connection utrace has to the mainline kernel, so proper review by the tracing folks and cooperation with the tracing folks is very much needed for the whole thing. 

But McGrath sees things rather differently. From his perspective, utrace has enough usefulness in its own right--not primarily as just a piece of SystemTap--to be considered for the mainline. Several different uses for utrace, in addition to the `ptrace()` cleanup, were mentioned in the thread: [kmview](<https://lwn.net/Articles/325295/>), a kernel module for virtualization; uprobes for DTrace-style user-space probing; changing UML to use utrace directly, rather than `ptrace()`; and more. Eigler also [defended](<https://lwn.net/Articles/325296/>) utrace as a standalone feature: 

utrace is a better way to perform user thread management than what is there now, and the utrace-ftrace widget shows how to *hook* thread events such as syscalls in a lighter weight / more managed way than the first one proposed. 

Molnar would like to see the "rewrite-ptrace-via-utrace" patch included before merging utrace. That would give the facility a solid in-kernel user, which could be used by other kernel developers to test and debug utrace. But, McGrath is not yet [ready to submit that code](<https://lwn.net/Articles/325297/>): 

The utrace-ptrace code there today is really not very nice to look at, and it's not ready for prime time. As has been mentioned, it is a "pure clean-up exercise". As such, it's not the top priority. It also didn't seem to me like much of an argument for merging utrace: "Look, more code and now it still does the same thing!" 

In some ways, the association with SystemTap is unfairly coloring the reaction to utrace. Molnar posted an excellent [summary](<https://lwn.net/Articles/325298/>) of the issues that stop him (and other kernel hackers) from using SystemTap--along with some possible solutions--but utrace and SystemTap aren't equivalent. It may not make sense to merge utrace without a serious in-kernel user of the interface, but most of the rest of the arguments have been about SystemTap, not utrace. As McGrath [puts it](<https://lwn.net/Articles/325300/>): 

This ptrace work really buys nothing with immediate pay-off at all. It's a real shame if its lack keeps people from actually looking at utrace itself. (This has been a long conversation so far with zero discussion of the code.) A collaboration with focus on what new things can be built, rather than on reasons not to let the foundations be poured, would be a lovely thing. 

It remains to be seen whether utrace will make its way into 2.6.30 or not. Linus Torvalds was [unimpressed](<https://lwn.net/Articles/325302/>) with utrace dominating Fedora kerneloops.org reports, as [relayed](<https://lwn.net/Articles/325309/>) by Molnar--though the bug that caused those problems has been long fixed. McGrath sees value in merging utrace before the `ptrace()` rewrite is ready, while other kernel developers do not. If utrace misses this merge window, it would seem likely that it will return for 2.6.31, along with the rewrite; at that point merging would seem quite likely.
