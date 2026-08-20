---
title: Minimizing instrumentation impacts
url: https://lwn.net/Articles/365833/
date: "December 9, 2009"
category: Jump label; Kprobes; Tracing
author: "By Jake Edge December 9, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
December 9, 2009

Minimizing the overhead of various kernel debugging and tracing mechanisms is important for many reasons. For static instrumentation, like tracepoints, the impact when they are not enabled must be very low or they won't get used--or merged. In addition, for any kind of instrumentation, the impact when enabled needs to be as small as possible so that whatever behavior is under observation will not radically change due to the tracing. Two separate proposals, [jump labels](<http://lwn.net/Articles/362752/>) for tracepoints and [kprobes jump optimization](<http://lwn.net/Articles/364189/>), are both trying to reduce the effect that instrumentation has on performance. In addition, they share some underlying code. 

The [kprobes](<http://lwn.net/Articles/343766/>) jump optimization has been proposed by Masami Hiramatsu, and trades off a bit of extra memory for approximately one-fifth the overhead in making a kprobe call. According to Hiramatsu's posting, kprobes went from 0.68us (32-bit) and 0.91us (64-bit) to 0.06us (both) when they were optimized with this technique. kretprobes dropped from 0.95us (32-bit) and 1.21us (64-bit) to 0.30 and 0.35us respectively. All of his testing was done on a 2.33GHz Xeon processor. 

Those numbers are pretty eye-opening, especially since the optimization only adds around 200 bytes per probe. The basic idea is to use a jump instruction, rather than a breakpoint, to implement probes whenever that is possible. The patch includes some fairly elaborate "safety checks" to see if it is possible to do the optimization. Before any of that is done, however, a regular breakpoint-based kprobe is inserted--if the optimization can't be done, that will be used instead. 

The jump instruction that will be put at the address to be probed is longer than one byte, so the optimization step needs to look at the region of code it will be affecting. If that region straddles the boundary between functions (i.e. spills out of the probed function into the next), the optimization is not done. It then decodes the function looking for jump instructions that would--or could--jump into the region, if none are found, the optimization proceeds. 

The instructions that are located at the address to be probed still need to be executed once they are replaced by a jump, of course, so a "detour" buffer is created. The detour buffer emulates an exception that contains the instructions copied from the probed location, followed by a jump back to the original execution path. This detour buffer will be used once the kprobe code itself is executed to finish the execution after the probe point. 

Once the detour buffer has been created, the kprobe is enqueued on the kprobe-optimizer workqueue, where the actual jump is patched into the probe site. The optimizer needs to ensure that there are no interrupts executing and does so by using `synchronize_sched()` in the workqueue function. Once that completes, the `text_poke_fixup()` function, which is added as part of the patchset, is called to actually modify the code to patch the jump in. 

The [`text_poke_fixup()` patch](<https://lwn.net/Articles/366007/>) is the piece that is shared with jump labels. It looks like: 

```
void *text_poke_fixup(void *addr, const void *opcode, size_t len,
                              void *fixup);
```

where `addr` points to the location to change, `opcode` and `len` specify the new opcode (and its length) to be written there. `fixup` is the address where a processor should jump if it hits `addr` while the modification is in process. 

Essentially, `text_poke_fixup()` puts a breakpoint that will execute the code at `fixup` on `addr` and synchronizes that on all CPUs. It then modifies all the other bytes (except the first) of the region, once again synchronizing with the other CPUs. The next step is to modify the first byte, again requiring synchronization, and then it can clear the breakpoint. Any calls made during the modification will be routed by the breakpoint to the `fixup` code instead. 

A jump label uses the same technique, but, since it applies to static instrumentation (tracepoints), it is meant to reduce the impact of the likely case that the tracepoint is disabled. It does that by using an assembly construct that will be available in the soon-to-be-released GCC 4.5, the `asm goto`, which allows branching to labels. 

For a tracepoint, the idea is that the disabled case will consist of a 5-byte NOP (conveniently sized to be overwritten with a jump) followed by a jump around the disabled tracepoint code. When the tracepoint gets enabled, `text_poke_fixup()` is used to turn the NOP into a jump to the label in the `DECLARE_TRACE()` macro. That code is what the original unconditional jump skips over. 

The jump labels patch then has code to manage the state of the tracepoints, including the labels and addresses, along with the current enabled/disabled status of the tracepoint. It is somewhat of a hackish abuse of the pre-processor and assembler, but according to Jason Baron, who proposed the patch, it results in ""an average improvement of 30 cycles per-tracepoint on x86_64 systems that I've tested"". 

Jump labels eliminate the current test and jump that is done for each tracepoint, because it can dynamically enable and disable the tracepoint code. Adding the NOP and unconditional jump add ""2 - 4 cycles on average vs. no tracepoint"", Baron [said](<https://lwn.net/Articles/366013/>), which is a pretty low cost for this kind of instrumentation. 

Both of these techniques are likely to need some more "soaking" time before they are ready for the mainline. Jump labels is a more recent proposal and relies on features in a not-yet-released compiler, which would seem to put it a bit further behind. The reaction to both has been relatively positive, though, which probably indicates general agreement with their goals. Reducing the overhead for tracing and debugging is something that few will argue against.
