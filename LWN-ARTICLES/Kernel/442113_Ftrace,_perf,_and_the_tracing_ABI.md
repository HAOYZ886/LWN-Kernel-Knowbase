---
title: "Ftrace, perf, and the tracing ABI"
url: https://lwn.net/Articles/442113/
date: "May 11, 2011"
category: "Development model-User-space ABI; Development tools-Kernel tracing; Ftrace; Performance monitoring; Tracing-ABI issues"
author: "By Jonathan Corbet May 11, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 11, 2011

Arjan van de Ven recently [reported](<https://lwn.net/Articles/442115/>) that a 2.6.39 change in how tracepoint data is reported by the kernel broke powertop; he requested that the change be partially reverted. The resulting discussion covered the familiar problem of how tracepoints mix with the kernel ABI. But it also revealed some serious disagreements on how tracing data should be provided by the kernel and, perhaps, the direction that this interface will take in the future. 

Each tracepoint defined in the kernel includes a number of fields containing values relevant to the specific event being documented. For example, the `sched_switch` tracepoint, which fires when the scheduler is switching between processes, includes the IDs of both processes, their priorities, and so on. Every tracepoint also has a few "common" fields, including the process ID, its flags, and the value of the `preempt_count` variable; if trace data is read in binary form, those values will appear at the beginning of the structure read from the kernel. 

Prior to the 2.6.32 development cycle, those common fields also included the thread group ID; that value was removed in September, 2009. A look at the [powertop source](<http://git.moblin.org/cgit.cgi/powertop/tree/perf.c>) shows that the program expects that field to still be there (though it does not use it); its internally-defined structure for trace data includes a `tgid` field. So this change should have broken powertop, and it would have except for one other change: on the very same day, Steve Rostedt added the `lock_depth` common field to report whether the current process held the big kernel lock (BKL). The addition of this field was never meant to be permanent: its whole purpose, after all, was to help with the removal of the BKL from the kernel entirely. 

For 2.6.39, the `lock_depth` common field was removed, and powertop broke. Arjan subsequently complained; he also supplied a patch which put a zero-filled padding field where `lock_depth` used to be. Steve [opposed the patch](<https://lwn.net/Articles/442121/>), on the grounds that, had powertop used the tracing ABI properly, it would not have broken. The kernel exports information about each tracepoint; for the above-mentioned `sched_switch`, that information can be examined from the command line: 

```
# cat /sys/kernel/debug/tracing/events/sched/sched_switch/format
        name: sched_switch
        ID: 51
        format:
    	field:unsigned short common_type; offset:0; size:2;	signed:0;
    	field:unsigned char common_flags; offset:2; size:1; signed:0;
    	field:unsigned char common_preempt_count; offset:3; size:1; signed:0;
    	field:int common_pid; offset:4; size:4; signed:1;
    
    	field:char prev_comm[16]; offset:8; size:16; signed:1;
    	field:pid_t prev_pid; offset:24; size:4; signed:1;
    	field:int prev_prio; offset:28; size:4; signed:1;
    	field:long prev_state; offset:32; size:8; signed:1;
    	field:char next_comm[16]; offset:40; size:16; signed:1;
    	field:pid_t next_pid; offset:56; size:4; signed:1;
    	field:int next_prio; offset:60; size:4; signed:1;
```

A properly-written program, Steve says, should read this file and use the offset values found there to obtain the data it is interested in. Linus [seemed to agree](<https://lwn.net/Articles/442122/>) that it would have been nice if things worked out that way, but that's not what happened. Instead, at least one program became dependent on the binary format of the trace data exported from the kernel. That is enough to make that format part of the kernel ABI; breaking that program counts as a regression. So Arjan's patch was merged. 

Steve did not like this outcome; it went against all the effort which had gone into creating a means by which tracepoints could change without breaking applications. The alternative, he [said](<https://lwn.net/Articles/442339/>), was to bury the kernel in compatibility cruft: 

The reason tracepoints have currently been stable is that kernel design changes do not happen often. But they do happen, and I foresee that in the future, the kernel will have a large number of "legacy tracepoints", and we will be stuck maintaining them forever. 

What happens if someone designs a tool that analyzes the XFS filesystem's 200+ tracepoints? Will all those tracepoints now become ABI? 

The notion that XFS tracepoints could become part of the ABI was [dismissed](<https://lwn.net/Articles/442340/>) as ""crazy talk"" by Dave Chinner, but there is nothing inherently different about those tracepoints. They could, indeed, end up as part of the kernel ABI. 

Steve was also concerned about the size of events; removal of `lock_depth`, beyond eliminating a (now) meaningless bit of data, also served to make each event four bytes smaller. There is always pressure to reduce the overhead of tracing, and reducing the bandwidth of the data copied to user space is part of that; adding the pad field goes against that goal. David Sharp (of Google) [chimed in](<https://lwn.net/Articles/442341/>) to note that data size matters a lot to them: 

The size of events is a *huge* issue for us. Please look at the patches we have been sending out for tracing: A lot of them are about reducing the size of events. Most of the patches we carry internally are about reducing the size of events. Memory is the most scarce resource on our systems, so we *cannot* afford to use large trace buffers. 

Steve had hoped to remove some of the other common fields as well (a change that Google has already made internally); that idea has gone by the wayside for now. Tracepoints, it seems, are ABI, even when the information they report no longer makes sense in the kernel. 

The remainder of this discussion became a sort of bunfight between Steve and Ingo Molnar as they sought to place the blame for this problem and to determine how things will go in the future. Ingo [attacked](<https://lwn.net/Articles/442124/>) Steve for resisting the idea of unchanging tracepoints, [accused](<https://lwn.net/Articles/442125/>) him of maintaining ftrace as a fork of perf in the kernel (despite the fact that ftrace was there first), and [said](<https://lwn.net/Articles/442126/>) that perf needed to take over: 

perf is basically the ftrace UI and APIs done better, cleaner and more robustly. Look at all the tooling that sprang up around that ABI, almost overnight. ftrace evolved through many iterations in the past and perf was simply the next logical step. 

He also threatened to stop pulling tracing changes from Steve. 

Steve, in return, blamed perf for bolting itself onto the ftrace infrastructure, then exporting ftrace's binary structures directly to user space. He blamed Ingo for blocking changes intended to improve the situation (for example, the creation of a separate directory for stable tracepoints agreed to at the [2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)) and complained that Ingo was ignoring his attempts to create tracing infrastructure which works for everybody. He also worried, again, that set-in-stone tracepoint formats would impede progress in the kernel. 

Despite all of this, Steve [is willing](<https://lwn.net/Articles/442129/>) to work toward the unification of ftrace and perf, as long as it doesn't mean leaving ftrace behind: 

Now that perf has entered the tracing field, I would be happy to bring the two together. But we disagree on how to do that. I will not drop ftrace totally just to work on perf. There's too many users of ftrace that want enhancements, and I will still support that. The reason being is that I honestly do not believe that perf can do what these users want anytime in the near future (if at all). I will not abandon a successful project just because you feel that it is a fork. 

So it seems that, while there are clearly disagreements and tension between the developers in this area, there should also be room for a solution that works for everybody. Development emphasis will clearly continue to move toward perf, but, despite Ingo's desire to the contrary, ftrace will likely continue to be improved. We may see efforts to push applications toward libraries that can shield them from tracepoint changes, but, for now, every tracepoint added to the kernel will probably have to be considered to be part of its ABI; given that, developers should probably be reviewing new tracepoints more closely than they have been. And, with luck, instrumentation in Linux - which has improved considerably in the last few years - will continue to get better.
