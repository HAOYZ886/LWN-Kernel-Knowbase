---
title: Triggers for tracing
url: https://lwn.net/Articles/556186/
date: "June 26, 2013"
category: "Development tools-Kernel tracing; Ftrace"
author: "By Jake Edge June 26, 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 26, 2013

The Ftrace kernel tracing facility has long had the ability to turn on or off tracing based on hitting a certain function (the trigger) during the trace. For 3.10, several new trigger commands that allow even more control over the tracing process were added, but can only be triggered by function entry. A recent [patch set](<https://lwn.net/Articles/555884/>) from Tom Zanussi would extend the Ftrace trigger feature to apply more widely so that the triggers could be specified for any tracepoint. Triggers allow the user to reduce the amount of tracing information gathered, so that narrowing in on the behavior of interest is easier. 

In 3.10, [patches](<https://lwn.net/Articles/543216/>) by Steven Rostedt add several trigger actions to Ftrace that will allow triggers to create a snapshot of the tracing buffers, add a stack trace to the buffers, or to disable and enable other trace events when the trigger is hit. Any of those (and the long-available traceon and traceoff trigger actions) can be associated with a particular function call in the kernel, but not with an arbitrary trace event. The latter is what Zanussi's patches would allow. 

Tracepoints can be placed by kernel developers in locations of interest for debugging or other kinds of monitoring; when an active tracepoint is hit while tracing is active, it will generate a trace event that gets stored in the tracing buffers. Some tracepoints have been added permanently to the kernel, but others can be placed _ad hoc_ to provide extra information, particularly when tracking down a bug. In contrast, the function entry events are placed automatically at build time when Ftrace is enabled in the kernel. In both cases, the tracepoints are disabled until a user enables them through the debugfs interface. 

Disabling or enabling a tracepoint is a fairly intrusive process that involves modifying the code at the tracepoint site. That, in turn, requires the use of `stop_machine()`, which will stop execution on all other CPUs in the system so that the modified code properly propagates to the entire system. But a trigger can fire on any function call (or, with Zanussi's patches, any tracepoint), so there needs to be a way to enable and disable tracepoints that doesn't require taking any locks or sleeping. That constraint led Rostedt to add a way to ["soft disable" trace events](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=417944c4c7a0f657158d0515f3b8e8c043fd788f>). Tracepoints are simply marked as disabled using a flag, rather than modifying the code itself, which means that the function handling the trace event will be called but will immediately return. Any tracepoints that might be enabled by a trigger are initialized at the time the trigger is set, but marked as soft disabled until the triggering event is hit. 

All of the trigger capabilities are aimed at narrowing in on a particular behavior (typically a bug) of interest. The pre-3.10 triggers only allowed using the "big hammer" of disabling and enabling tracing itself, while these new trigger actions provide more fine-grained control. For example, using the "snapshot" trigger action will create a snapshot of the existing tracing buffer in a separate file while continuing the trace. Rostedt shows an example in his [update to `Documentation/trace/ftrace.txt`](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=8d016091d10953e00f9d2c0125cc0ddd46c23a6a>): 

```
# in the tracing debugfs directory, typically /sys/kernel/debug/tracing
        echo 'native_flush_tlb_others:snapshot:1' > set_ftrace_filter
```

That command will trigger a snapshot, once (that's what the "`:1`" does), when the `native_flush_tlb_others()` function is called. 

Or to enable a specific event: 

```
echo 'try_to_wake_up:enable_event:sched:sched_switch:2' > set_ftrace_filter
```

That will enable the `sched_switch` event in the `sched` subsystem when the `try_to_wake_up()` function is called. The event will be enabled the first two times that `try_to_wake_up()` is called, presumably some other trigger is disabling the event in between. 

Zanussi's patches (which come with good documentation and examples, both in the introductory patch and an [update to `Documentation/trace/events.txt`](<https://lwn.net/Articles/556254/>)) apply those same triggers to the events interface. Thus the `kmem:kmalloc` event can be enabled when the `read()` system call is entered as follows: 

```
# again from /sys/kernel/debug/tracing
        echo 'enable_event:kmem:kmalloc:1' > events/syscalls/sys_enter_read/trigger
```

That will also cause the event to be soft disabled until the trigger is hit, which can be verified by: 

```
cat events/kmem/kmalloc/enable
```

That will show a "0*" as output, which means that the event is soft disabled. 

But there is more that can be done using triggers. The trace events filter interface allows tests to be performed to filter the tracing output, and Zanussi's patches extend that to the trigger feature. So adding a stack backtrace to the tracing buffer for the first five `kmem:kmalloc` events where the bytes requested value is 512 or more could be done this way: 

```
echo 'stacktrace:5 if bytes_req >= 512' > events/kmem/kmalloc/trigger
```

One can check on the status of the trigger using: 

```
cat events/kmem/kmalloc/trigger
```

which would show: 

```
stacktrace:count=2 if bytes_req >= 512
```

if three of the five stack backtraces had been emitted into the tracing buffer (`count=2`). 

The triggers and tests can be mixed and matched in various ways, but there are some restrictions. There can only be one stacktrace or snapshot trigger per event and each event can only have one traceon and traceoff trigger. A triggering event can have multiple enable_event and disable_event triggers, but each must refer to a different event to be enabled or disabled. So `sys_enter_read` could enable two different events (with two different commands echoed into its `trigger` file) but the events so enabled must be distinct. 

The Ftrace triggers will be released soon with 3.10. Zanussi's patches are still in the review stage. Given that the 3.11 merge window will likely open soon, triggers for all events will probably have to wait for 3.12.
