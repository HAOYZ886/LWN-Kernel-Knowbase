---
title: "Heuristics for software-interrupt processing"
url: https://lwn.net/Articles/925540/
date: "March 13, 2023"
category: "Interrupts-Software"
author: "By Jonathan Corbet March 13, 2023"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 13, 2023

The kernel's software-interrupt ("softirq") mechanism was added prior to the 1.0 kernel release, but it implements a design seen in systems that were already old when Linux was born. For much of that time, softirqs have been an impediment to the kernel community's scalability and response-time goals, but they have proved resistant to removal. A recent discussion on a proposed new heuristic to mitigate a softirq-related performance problem may have reinvigorated interest in doing something about this subsystem as a whole rather than just tweaking the parameters of how it operates. 

Hardware interrupts are generated when some component of the system needs the CPU's attention to, for example, deal with a completed I/O operation. The processing of hardware interrupts is one of the highest-priority tasks in the kernel; an interrupt will preempt almost anything else that might be running, so the amount of work done in interrupt handlers must be kept to a minimum to avoid adversely affecting the rest of the system. The softirq mechanism was designed to allow hardware-interrupt handlers to set aside work to be done urgently — but not quite as urgently as hardware-interrupt processing. 

The subsystems that use software interrupts include networking, timers, the block subsystem, read-copy-update (RCU), and tasklets. When one of these subsystems has work to delegate to a softirq handler, it will "raise" a softirq by setting a bit in a special mask. When the kernel is in a position to handle software interrupts — usually either at the end of hardware-interrupt processing or on return from a system call — it will make a pass over the raised softirqs and call the appropriate handler for each. 

In practice, a softirq raised out of a hardware-interrupt handler will often be run immediately after that hardware handler finishes, but that is not necessarily the case. Softirqs can also be raised out of any (kernel) context, not just while responding to hardware interrupts; the RCU softirq, for example, is not tied to any hardware interrupt at all. 

The problem here is that there may be a lot of work for the softirq handlers to do. They are invoked for, among other things, packets received from the network and RCU callbacks and, by the time the handlers run, there may be thousands of each waiting. So softirq processing can go on for a long time, to the detriment of the rest of the work the system is meant to be doing. This problem gets worse if, as can easily happen, _more_ work shows up while softirq handling is happening. 

#### Managing softirq handling

To avoid overwhelming the system with softirq processing, a number of heuristic mechanisms have been added to the kernel over time. These include: 

  * The function that normally processes software interrupts ([`__do_softirq()`](<https://elixir.bootlin.com/linux/latest/source/kernel/softirq.c#L528>)) will pass over all of the raised softirqs and process them. Once that is done, it checks if more softirqs are pending; should that be the case, it will go back to the beginning — but only for a maximum of ten times. If that count is exceeded, the kernel stops processing softirqs and, instead, wakes the per-CPU `ksoftirqd` kernel thread to continue the work. 
  * Similarly, if softirq processing continues for more than (approximately) 2ms, the remaining work will be punted to `ksoftirqd`. 
  * Whenever the `ksoftirqd` thread is running on a given CPU, the kernel will not even try to process software interrupts there; it will just leave them for the thread to handle. 
  * When the kernel _is_ processing software interrupts, it will occasionally check the current process (the one that was preempted to handle softirqs) for the `TIF_NEED_RESCHED` flag, which indicates that a higher-priority process is ready to run. In that case, it will stop processing softirqs to defer to that process. 

Some of the subsystems using software interrupts also implement limits in their own handlers, independent of anything that the central softirq code manages (or even knows about). 

Toward the end of December, Jakub Kicinski posted [a patch set](<https://lwn.net/ml/linux-kernel/20221222221244.1290833-1-kuba@kernel.org/>) (under the title ""uncontroversial change"") addressing a problem that has been encountered with the last heuristic listed above. The deferral of softirq processing when the kernel wants to reschedule was meant to allow low-latency processing of events. If some audio data comes in, for example, a recorder application will want be able to run and to grab it quickly. But if the process that runs after reschedule is not quick — if it holds onto the CPU for a long time — it will block all softirq processing for that long time. That can cause undesirable behavior like networking stalls, TCP retransmissions, and more. 

To fix the problem, Kicinski has proposed the addition of another heuristic. Once the rescheduling deferral happens, the kernel will only wait 2ms before it starts handling softirqs again, regardless of whether ksoftirqd is running; this will keep a long-running process from blocking softirq processing for too long. A similar timeout applies to deferrals caused by an overload of softirqs — the first two items in the list above. In that case, the kernel will restart handling softirqs after 100ms. These changes, he said, result in a tenfold reduction in networking stalls and a 50% drop in TCP retransmissions. 

#### Addressing the real problem

Thomas Gleixner [seemed willing to accept the new timeouts](<https://lwn.net/ml/linux-kernel/87r0u6j721.ffs@tglx/>), though he added that he is ""not a big fan of heuristical duct tape"". He pointed out some problems with the timekeeping; it is using the `jiffies` time variable, which has a resolution of 1-10ms, for small-millisecond values. That can lead to fairly widely varying results — a problem that, [he later realized](<https://lwn.net/ml/linux-kernel/87jzzxkgno.ffs@tglx/>), exists in current kernels as well. But it didn't take much longer before he [complained about the whole approach](<https://lwn.net/ml/linux-kernel/87r0u3hqtw.ffs@tglx/>) which, he said, just makes the overall softirq problem worse. 

Softirqs, he said, ""are just the proliferation of an at least 50 years old OS design paradigm"". They exist to allow certain code to circumvent whatever resource management (CPU scheduling, for example) is in place and, as a result, make it impossible to fully control the operation of the system. He grumbled that the usual approach when problems inevitably turn up is to add more heuristics and knobs. ""Can we please stop twiddling a parameter here and there and go and fix this whole problem space properly"?", he asked. He also acknowledged that he did not have a complete solution in mind, but promised to think about it in the near future. 

Kicinski [said](<https://lwn.net/ml/linux-kernel/20230306165131.11b7b475@kernel.org/>) that the networking developers are trying to move processing out of software interrupts, but that it is a long, slow process. Frederic Weisbecker, instead, [pointed out](<https://lwn.net/ml/linux-kernel/ZAXVF0tPKLErAkpT@lothringen/>) the real problem with softirqs: only one of them can run at a time, and there is kernel code that depends on that level of exclusion. Nobody really knows, after all these years, which softirq-handling code can safely run concurrently with other handlers, and which cannot. So disabling of software-interrupt processing has taken on a role similar to that of the unlamented big kernel lock and, like the big kernel lock, softirqs are hard to safely get rid of. 

The solution, he suggested, is a pushdown effort similar to [what was done for the big kernel lock](<https://lwn.net/Articles/283066/>). All softirq processing could be pushed into kernel threads, which would then be scheduled in the normal way, but they would all start by running with the equivalent of the softirq-disable big lock held. As the handlers for specific subsystems were proven safe, they could release that lock until, someday, it would wither away entirely. The task, he said, is not for the faint of heart: 

> Of course this is going to be a tremendous amount of work but it has the advantage of being iterative and it will pay in the long run. Also I'm confident that the hottest places will be handled quickly. And most of them are likely to be in core networking code. 

All that is needed is somebody to actually do all of that work. 

Matthew Garrett once [said](<https://lwn.net/Articles/461720/>) that ""'heuristic' is an ancient African word meaning 'maybe bonghits will make this problem more tractable'"". Certainly software interrupts have been the sort of intractable problem that can drive developers to such a remedy. Perhaps, someday, this ancient subsystem will be cleaned out, urgent processing will be done in a controlled manner, and the heuristics will no longer be needed. For now, though, they are still necessary, and Kicinski's patch may be the sort of bandage that makes them work a little better for a little longer while the real problem is being solved. After all, even though the development community will surely deal with softirqs for real this time, this work is likely to take a while to come to fruition.
