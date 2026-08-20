---
title: Accurate timestamps for the ftrace ring buffer
url: https://lwn.net/Articles/831207/
date: "September 22, 2020"
category: "Ftrace; Tracing-Ftrace"
author: "September 22, 2020 This article was contributed by Steven Rostedt"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

September 22, 2020

This article was contributed by Steven Rostedt

The [function tracer (ftrace) subsystem](<https://www.kernel.org/doc/html/v5.8/trace/ftrace.html>) has become an essential part of the kernel's introspection tooling. Like many kernel subsystems, ftrace uses a [ring buffer](<https://en.wikipedia.org/wiki/Circular_buffer>) to quickly communicate events to user space; those events include a timestamp to indicate when they occurred. Until recently, the design of the ring buffer has led to the creation of inaccurate timestamps when events are generated from interrupt handlers. That problem has now been solved; read on for an in-depth discussion of how this issue came about and the form of its solution. 

#### The ftrace ring buffer

The ftrace ring buffer was [ added in 2008](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7a8e76a3829f1067b70f715771ff88baf2fbf3c3>) and, a little less than a year later, it became [completely lockless](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=77ae365eca895061c8bf2b2e3ae1d9ea62869739>). The [design of the ring buffer](<https://www.kernel.org/doc/html/v5.9-rc4/trace/ring-buffer-design.html>) split it into per-CPU buffers; each per-CPU buffer has a series of sub-buffers, the size of which happens to be the architecture's page size. This sizing was not a requirement of the design, but it is a convenient size for the `splice()` system call. Each sub-buffer begins with a header that includes, among other things, a full timestamp for the first event stored there. 

Writes to a specific per-CPU buffer can only happen on the CPU for that buffer. That ensures that any contention between writers will always be in stack order. That is: a write being done in normal context could only have a contending writer running in an interrupt context, and that write must completely finish before returning back to normal context. There is no need to worry about parallel writers that are executing on other CPUs. Interrupted writes will thus always appear as shown below: 

> ![\[Interrupted write timing diagram\]](https://static.lwn.net/images/2020/sr/interrupt.png)

The design of the ring buffer depends on the fact that writers that interrupt other writers will completely finish before the interrupted writer may continue. This allows for some flexibility in how the writers can remain lockless. Although this simplified the coordination between writes, it added extra complexity to the tracking of time. 

Before going into timestamp management, an understanding of how space is reserved on the ring buffer is necessary. An index is used to denote where the last event in the sub-buffer was written. The length of each new event is added to the index with `local_add_return()` (which can be used since this is a per-CPU index) and the location for the new event is simply the returned value minus the length of the event. 

> ![\[Reserving space for an event in the ring buffer\]](https://static.lwn.net/images/2020/sr/local-add.png)

Obviously, if the value returned is greater than the size of the sub-buffer, it means there's no more room on the sub-buffer for this event, and the logic to [move to the next page in the ring buffer](<https://lwn.net/Articles/340400/>) is invoked. 

#### Timestamps

A 64-bit timestamp requires eight bytes to store. The bigger an event is, the longer it takes to write it and the fewer of them a ring buffer may hold. To keep the event header small, the ring buffer code tries to avoid storing the full timestamp. An event on the ring buffer looks like this:

```
struct ring_buffer_event {
    	u32 type_len:5, time_delta:27;
    	u32 array[];
        }
```

The first five bits of the event header determines its type and size, where a value of 29 means it is a padding event, 30 is a time-extend event, and 31 is an absolute timestamp. If the value is between one and 28, it represents an event with a data payload that starts at the `array` field, and the total event size is the `type_len` times four. If the total event size is greater than 112 (or 4*28) bytes, then `type_len` is set to zero, the 32-bit `array` field will hold the length of the event, and the data payload starts immediately after the `array` field. With most events having a size of 112 bytes or less, this helps keep the events compacted. Note that all events are four-byte aligned. 

The next 27 bits of the first integer of the event is the `time_delta`. This field holds the delta of time since the last event (or zero if it's the first event on the sub-buffer, which holds a full eight-byte timestamp in its header). If the timestamp is in nanoseconds, the largest delta that can be stored is 134.217728 milliseconds (227 nanoseconds). If an event comes in after 134.217728 milliseconds, a time-extend event is added, which uses both the `time_delta` and the 32 bits of the `array` field to create a delta of up to 18 years (259 nanoseconds). 

Tom Zanussi needed full timestamps from the events at the time they were recorded to get [histograms](<https://lwn.net/Articles/635522/>) to work. As the events only held deltas, a new event was created to store 59 bits of the full timestamp since boot up to allow the histograms to store the exact timestamp used for an event. The type 31 was used to denote this new event, which has the same make up as a time extend but, instead of being a delta, it would hold the time since boot. In actuality, this new time stamp could replace time extends since they could only fail if a machine was running for over 18 years without a reboot. 

#### The problem with nested writes

Using a delta from the previous event proved to be a troublesome design; it requires saving the timestamp of the last event written into the ring buffer for use in calculating the delta stored in the next event. This put several actions in play that need to be atomic but cannot be: 

  1. Reading the timestamp to use for the current event.
  2. Reserving a spot on the ring buffer to store the current event.
  3. Calculating the delta of the timestamp of the current event from the timestamp of the previous event.
  4. Saving the timestamp used for the current event to calculate the delta for the next event.

Any of the above steps can be interrupted by another context, such as an interrupt or non-maskable interrupt (NMI). This makes it difficult to know if the delta stored for the current event was really the delta since the timestamp of the previous event. After the last timestamp is retrieved for the delta calculation, an interrupt may occur and several events may be injected into the ring buffer before storage is allocated for the current event: 

> ![\[Interrupt during event storage allocation\]](https://static.lwn.net/images/2020/sr/interrupt-time.png)

The timestamp for the new event must be taken before the allocation, so that it can be used to calculate deltas for events that may come in via an interrupt that occurs right after the storage was allocated. Even if a full timestamp were written for the interrupt events, the timestamp used for the interrupted event, if retrieved after the space allocation, would be later than the interrupt-event timestamps, even though the interrupted event itself happened first, as shown below: 

> ![\[Timestamps taken after storage allocation\]](https://static.lwn.net/images/2020/sr/interrupt-time-before.png)

Regardless of whether the timestamp is taken before or after the allocation is performed, the interrupt situations described above will cause time to appear to go backward in the ring buffer. That is considered unacceptable because it breaks the merge sort used when all of the per-CPU buffers are shown together as a single output. 

The approach taken to avoid this problem was simply to write a zero delta for events that interrupt the writing of another event. Unfortunately, this makes it look as if time stood still. The obvious problem with this approach is that you lose the time between events when they interrupted the writing of another event. The output will look like all the events happened instantaneously. This approach has been satisfactory for the last 12 years, but it was a design flaw that needed to be fixed. 

To see this problem in real use, try running a command like: 

```
trace-cmd record -p function
```

for a while and then running: 

```
trace-cmd report --debug -l -t --ts-diff --cpu 4
```

on the output file. Here, `--debug` shows where the sub-buffer breaks are, `-l` shows latency information (interrupt context), `-t` keeps the timestamps in nanosecond format (otherwise it will truncate to microseconds), `--ts-diff` shows the delta between events, and `--cpu 4` is used just because I found what I was looking for on CPU 4 (I searched for the time delta of zero). This gives a good idea of the impact of what happens when events occur after interrupting the writing of another event. 

```
trace-cm-1724    4....   137.210588990: (+84)    function:                      kfree [84:0xf44:24]
        trace-cm-1724    4....   137.210589078: (+88)    function:                      wakeup_pipe_writers [88:0xf60:24]
        trace-cm-1724    4d.h.   137.210589709: (+631)   function:             __sysvec_apic_timer_interrupt [631:0xf7c:24]
        trace-cm-1724    4d.h.   137.210589709: (+0)     function:                hrtimer_interrupt [0:0xf98:24]
        trace-cm-1724    4d.h.   137.210589709: (+0)     function:                   _raw_spin_lock_irqsave [0:0xfb4:24]
        trace-cm-1724    4d.h.   137.210589709: (+0)     function:                   ktime_get_update_offsets_now [0:0xfd0:24]
        CPU:4 [SUBBUFFER START] [137210590461:0x27c53000]
        trace-cm-1724    4d.h.   137.210590461: (+752)   function:                   __hrtimer_run_queues [0:0x10:24]
        trace-cm-1724    4d.h.   137.210590461: (+0)     function:                      __remove_hrtimer [0:0x2c:24]
        trace-cm-1724    4d.h.   137.210590461: (+0)     function:                      _raw_spin_unlock_irqrestore [0:0x48:24]
        trace-cm-1724    4d.h.   137.210590461: (+0)     function:                      tick_sched_timer [0:0x64:24]
        trace-cm-1724    4d.h.   137.210590461: (+0)     function:                         ktime_get [0:0x80:24]
        trace-cm-1724    4d.h.   137.210590461: (+0)     function:                         tick_sched_do_timer [0:0x9c:24]
        trace-cm-1724    4d.h.   137.210590461: (+0)     function:                         tick_sched_handle.isra.0 [0:0xb8:24]
        trace-cm-1724    4d.h.   137.210590461: (+0)     function:                            update_process_times [0:0xd4:24]
        [...]
        trace-cm-1724    4d.s.   137.210590461: (+0)     function:                   rcu_segcblist_pend_cbs [0:0x940:24]
        trace-cm-1724    4d.s.   137.210590461: (+0)     function:                rcu_disable_urgency_upon_qs [0:0x95c:24]
        trace-cm-1724    4d.s.   137.210590461: (+0)     function:                rcu_report_qs_rnp [0:0x978:24]
        trace-cm-1724    4d.s.   137.210590461: (+0)     function:                _raw_spin_unlock_irqrestore [0:0x994:24]
        trace-cm-1724    4..s.   137.210590461: (+0)     function:                rcu_segcblist_ready_cbs [0:0x9b0:24]
        trace-cm-1724    4d.s.   137.210590461: (+0)     function:             irqtime_account_irq [0:0x9cc:24]
        trace-cm-1724    4....   137.210590461: (+0)     function:             kill_fasync [0:0x9e8:24]
        trace-cm-1724    4....   137.210605019: (+14558) function:             pipe_unlock [14558:0xa04:24]
        trace-cm-1724    4....   137.210606026: (+700)   function:             __x64_sys_splice [700:0xa58:24]
```

Looking at this output, I can tell that the call to `__sysvec_apic_timer_interrupt()` happened from an interrupt that came in as the call to `kill_fasync()` started to be recorded but before it reserved space on the ring buffer. I know this because `__sysvec_apic_timer_interrupt()` has a time delta, thus it was able to reserve space on the ring buffer before `kill_fasync()` was able to, but after the processing of the event for `kill_fasync()` started. Once the processing of events happen, only the first event to get on the ring buffer will have a delta timestamp, then all events after that (including the one that was interrupted because its storage comes later), gets a zero delta. 

The `--debug` option for `trace-cmd report` is what caused the extra data to show in the output, which includes this line: 

```
CPU:4 [SUBBUFFER START] [137210590461:0x27c53000]
```

This output indicates that the trace crossed over a sub-buffer page at this point. As each sub-buffer stores an absolute timestamp, the first event on the sub-buffer will also have a delta as shown above. 

Over the years, this flaw really bothered me; I would spend countless hours thinking about how to find a way to reliably make the nested timestamps meaningful. The fact that we only needed to worry about stacked writes and not concurrent writes made me believe there was a solution. As there are only realistically four levels of the stack to worry about, I thought I could make a state for each level and use the above and below states to synchronize the timestamps. Those four levels are: normal context, software-interrupt (softirq) context, interrupt context, and NMI context. 

Theoretically, you could have a machine check during an NMI, making a fifth level, but odds of a softirq interrupting the writing of an event, and it too writing an event that gets interrupted by an interrupt, that then writes an event, where an NMI were to trigger and it too were to write an event is extremely low, and to put a machine check on top of that was even lower. Even with running function tracing that traces every function in all contexts, I had trouble finding one nested level, let alone four levels. And we could detect the nesting level, so the worst that could happen is that we store zero for the delta on detecting it. This turned out not to be a worry as my solution does not need to know about the levels. 

#### Avoiding `cmpxchg()`

In all my prior attempts to solve this problem, I tried hard to avoid the use of `local_cmpxchg()` (the local CPU version of `cmpxchg()`). `cmpxchg()` is an architecture-specific function that will atomically read a value from a given location, compare it with a given value and, if the two are equal, it will write a third value back to that location. If the values do not match, then the location is not updated. The original value read from the location is the return value of the `cmpxchg()`; it can be used to determine if the `cmpxchg()` succeeded in updating the location or not. 

When I first started working on the ring buffer, all of my benchmarks would show a slight but noticeable overhead when using `local_cmpxchg()` over `local_add_return()`. The goal was thus to not use a `cmpxchg()` and have, instead, a timestamp that would be used for each level of nesting. Starting with a four-element array of timestamps, I tried various approaches of a nesting counter and storing timestamps in each level. Upon detecting nesting, I thought that a context that interrupted another context could fix up the timestamps of the contexts that were interrupted without needing `local_cmpxchg()`. But this became much more complex, and had to deal with issues like this: 

> ![\[Three levels of interrupts\]](https://static.lwn.net/images/2020/sr/interrupt-3-level.png)

Having to deal with an array of timestamps just added one more variable that needed to be synchronized with the other variables. 

The above figure shows a case where an interrupt comes in right after the timestamp was taken and the storage was allocated for the first event, but before the event is actually stored. Then an NMI comes in after the timestamp and storage is allocated for an event happening in interrupt context. At this point, because the allocation during the NMI would not be the first event in the commit, and because two other contexts were interrupted below, it is difficult to know if it should update the timestamp of the event that happened in the interrupt context or not; the timestamp may have already been updated. On top of this, another event is recorded in interrupt context after the NMI added an event, and the state for this event would have to deal with an event injected from another context since the previous event recorded in the interrupt context. The number of states that are added by keeping track of four levels of context and how they relate to inserting events into the ring buffer grew so numerous that it became obvious this was not going to be a viable solution. 

#### The twelve year old puzzle solved

Julia Lawall reported a bug where she recorded a trace with [trace-cmd](<https://trace-cmd.org>) and found that time went backward. Looking into it, I discovered that it was due to the addition of the full timestamp used by Zanussi's histograms; the change allowed the time extensions to not be reset to zero if they occurred in a nested event. Writing the [fix](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.9-rc1&id=097350d1c6e1f5808cae142006f18a0bbc57018d>) for that issue triggered another idea for solving the nested timestamp issue. 

All my previous attempts tried to avoid using `cmpxchg()`. While debugging the issue that Lawall reported, I realized that nested events were extremely uncommon and, because they can be detected, it should be possible to separate the slow path from the fast path. A fast path is the common case, which is when an event being written did not interrupt another event, and also was not interrupted itself. Otherwise the slow path is run. `cmpxchg()` should not be a performance problem if it were only to be used in the slow path. Not restricting what can be done in the slow path allowed me to think about other possible solutions. This gave me new hope, and inspired me to look for a solution in this direction. 

While incorporating `cmpxchg()` back into the solution, I found that the array of four states still added too much complexity. I looked into whether it would be possible to consolidate the array, and only care about an event that interrupts another event or the event being interrupted. Upon interrupting an event in a lower context, it is known that the interrupted event is, in essence, "frozen in time". That is, it will not proceed until the current context returns to it. For being interrupted, there are only two states; before being interrupted and after being interrupted. What that means is that once interrupted, when processing resumes, everything that happened in the interrupt would have come to a conclusion. With the above characteristics, a defined set of states can be calculated for every step of the algorithm, by keeping track of two different timestamps: one that happens before allocating storage on the ring buffer, and one that happens afterward. 

Thus, the solution deals with three players: 

  * `write_tail`: the index used to reserve space on the buffer for the event.
  * `before_stamp`: a timestamp saved by all events as they start the recording process.
  * `write_stamp`: a timestamp updated after an event has successfully reserved space.

The following code is run in this order to determine the next decisions to be made: 

```
w = local_read(write_tail);
        before = local_read(before_stamp);
        after = local_read(write_stamp);
        ts = clock();
```

Before doing anything else, this code saves the current value of `write_tail` for later use. At this point, we can decide whether this event needs to go into the slow path or not. If `before` does not equal `after`, one of two possibilities is indicated: this event interrupted another event while it was updating its timestamps, or this event was interrupted by another context after reading `before_stamp` and before reading the `write_stamp`. In either case, the code would fall into the slow path. 

```
if (before != after) {
    	event_length += RB_LEN_TIME_EXTEND;
    	add_timestamp = RB_ADD_STAMP_FORCE | RB_ADD_STAMP_EXTEND;
        } else {
```

One part of this solution requires injecting absolute timestamps instead of using a delta. For this slow path, the event length is increased by the size of the absolute timestamp event (which is the same size as a time extend). The `ADD_STAMP_FORCE` and `ADD_STAMP_EXTEND` flags are saved for later use in the algorithm. 

Even if this event did not interrupt another event, a check still must be made to see if the delta since the last event stored can fit in the `time_delta` portion of the event. Otherwise a time extend is required. 

```
delta = ts - after;
    	if (delta & ~((1 << 27) - 1)) {
    	    event_length += RB_LEN_TIME_EXTEND;
    	    add_timestamp = RB_ADD_STAMP_EXTEND;
    	}
        }
```

Now write to the `before_stamp` and allocate storage on the ring buffer by adding to the `write_tail`. 

```
local_set(before_stamp, ts);
        write = local_add_return(event_length, write_tail);
        tail = write - event_length;
```

The start of the event can be found by subtracting its length from `write`, which is the index of the end of the event. This is stored in `tail`. Now compare the saved `write_tail` from the start of this algorithm with the calculated value of the start of the event. If they match, we know that no event interrupted this algorithm between the saving of `write_tail` and the allocation of the storage for the event. This is the fast path. But we are not out of the woods yet. We still need to update the `write_stamp`. Note, that the `before_stamp` has already been updated, making it different than the `write_stamp`. Any nested event that interrupts this event will now fall into the slow path and use an absolute timestamp. 

The next step is to simply update the `write_stamp`: 

```
local_set(write_stamp, ts);
```

But wait! What if an interrupt came in just before writing into the `write_stamp` and that interrupt wrote an event? Wouldn't that cause the `write_stamp` to now be incorrect, as it would not contain the timestamp of the last event written to the ring buffer? The answer is yes, but we don't care. The reason is that `write_stamp` is not used for any calculation unless it equals `before_stamp`. Because `before_stamp` does not equal `write_stamp`, any nesting events will not use it for their calculations. 

This is how stacked interrupting events (where all interrupting events finish before this event can continue) helps in our algorithm. `before_stamp` is always updated by all events, including nested events that interrupted this event, so the contents of `before_stamp` now contains the timestamp representing the last event stored in the ring buffer, and is what `write_stamp` also needs to be set to. Updating `write_stamp` still needs some care, but it is still easy to detect if this event was interrupted by another, and if so, the slow path is entered, and `cmpxchg()` can be taken advantage of: 

```
save_before = local_read(before_stamp);
    
        if (add_timestamp & RB_ADD_STAMP_FORCE)
    	delta = ts; // will use the full timestamp
        else
    	delta = ts - after; // remember, not force means not nested
    
        if (ts != save_before) {
    	after = local_read(write_stamp);
    	if (save_before > after)
    	    local_cmpxchg(write_stamp, after, save_before);
        }
```

The above code first re-reads `before_stamp`; it runs after `write_stamp` was updated. If another event came in between reserving space for the buffer and updating `write_stamp`, then `before_stamp` will not equal the read timestamp (`ts`). If the timestamp is still equal to `before_stamp`, then `write_stamp` was updated without worrying about racing with other interrupting events. At this point, the delta for the event is also calculated. If the `ADD_STAMP_FORCE` flag is set, that means this event interrupted another event and an absolute timestamp is required. Otherwise, it is safe to calculate the delta from the `write_stamp` and the clock value that was read. 

If `before_stamp` is not equal to the read clock (`ts`), that tells us that an event came in and updated `before_stamp` as well as `write_stamp` sometime after the storage for this event was allocated (the update of `write_tail`). As there is no way of knowing when that occurred, it must be assumed that it could have occurred before the update to `write_stamp`. To solve this, a simple `cmpxchg()` is performed by re-reading `write_stamp`; if `write_stamp` is less than the last read `before_stamp` then we have to update it. If `write_stamp` is greater than or equal to the last read `before_stamp` or the `cmpxchg()` fails, then there is nothing to be done. That is because this can only happen if this event was interrupted by another event after the update to `write_stamp` and that nested event would have taken care of the correctness of `write_stamp`. 

This is the end of the algorithm for the case of not being interrupted between taking the timestamps and allocating space on the ring buffer. But what happens if this event was interrupted before the allocation of its space on the ring buffer? 

#### The case of the interrupted event before allocating storage

In this path, an interrupt came in and other events were injected into the ring buffer somewhere between the first read of the `write_tail` and reserving space on the ring buffer for this event. At this moment, nothing can be trusted. Some work needs to be done to get back to some kind of known state. 

```
after = local_read(write_stamp);
        ts = clock();
```

As this event was interrupted and nested events made it into the ring buffer, the original recording of the clock (`ts`) is useless. Also, because this is the path of being interrupted by another event, the nested event (or events) would make sure that the `write_stamp` is the timestamp of the last event added to the ring buffer. Thus we reread both the clock and `write_stamp` to get into some kind of known state. 

```
if (write == local_read(write_tail) && after < ts) {
    	delta = ts - after;
    	local_cmpxchg(write_stamp, after, ts);
```

**Note** : the above code turns out to contain a subtle bug; the author will accept the first patch containing a correct fix. First-timers should review [this document](<https://www.kernel.org/doc/html/latest/process/submitting-patches.html>) describing how to submit patches to the kernel before sending. 

If `write_tail` equals the value returned by the `local_add_return()` when allocating this event, then there was no nested event occurring after the allocation and the re-reading of `write_stamp`. This means that this event is the last event in the ring buffer, and `write_stamp` needs to be set to it. A `cmpxchg()` is used to update `write_stamp` only if it hasn't changed. If a nested event came in after reading the `write_stamp` then that event would be the last event on the ring buffer, and the `write_stamp` should not be updated (the `cmpxchg()` would fail and nothing more needs to be done.). The delta can safely be calculated as it is known that the `write_stamp` is from the event stored just before this event was allocated. 

```
} else {
    	delta = 0;
        }
```

If the value returned by `local_add_return` does not match `write_tail`, that means an interrupt came in between the allocation of this event and the re-reading of `write_tail`. In this case this event was recorded between two interrupts that had nested events. One before the event was allocated, and again after it was allocated on the buffer. As there is now no way to know what timestamp to use for calculating its delta, we have no choice but to go back to a zero delta, but this is actually the best thing to do. If this event was sandwiched between two sets of events, what timestamp it has really does not matter in any use case, as long as it is shown to have happened between the two sets of nested events. 

> ![\[An event sandwiched between interrupts\]](https://static.lwn.net/images/2020/sr/interrupt-sandwich.png)

One might think that the above code is a little ambitious, and it may be fine to simply use a zero delta if an interrupt happens between the start of the processing and the allocation of the event. Why not just set the delta to zero in this case? The reasoning behind that is because it is not that uncommon of a case to hit. While tracing several hackbench runs, this situation happens a few times. The problem with just using zero for a delta is that, if the event recorded in the interrupt happened at the start of the interrupt, and the interrupt itself ran for some time before returning, then the zero delta would make it seem that the interrupt was much shorter than it actually was. 

But doesn't that excuse make any delta zero a problem? Unfortunately, yes. But the case of being interrupted by two different interrupts just before and just after storage is highly unlikely. It may still happen, but as stated, there's not much we can do about it. After running several traces of hackbench, I could not find a single occurrence of that happening. The only way I was able to test this last code path was by artificially injecting an event in a "fake" context and seeing if the algorithm performed as expected. 

At this point the problem has been solved — on 64-bit systems. It turns out that there was an additional obstacle to overcome for the 32-bit case; those looking for the details can find them in [this supplemental article](<https://lwn.net/Articles/831892/>). 

#### Conclusion

For several years I was afraid that correct timestamps for ftrace ring-buffer events would end up being impossible for a Turing machine to achieve. But as I agonized over the zero-delta flaw, I knew I had only two options to make the pain go away. Prove that it is an impossible solution and walk away from it with my tail between my legs, or find a solution that actually works. The first was not an option, as I also know that impossible problems can have possible solutions if you can put restrictions on the requirements. For instance, we still have one zero-delta path. But that path is so uncommon, and only affects a single event, thus it is not worth agonizing over. 

What I felt was most interesting from this experience, was that my solution was the least complex of those that I tried. That shouldn't be surprising. A lot of problems never get solved because people tend to overthink the solutions. All it took for me was to debug something slightly related to the issue to help me not overthink it as much, and everything just fell into place after that.
