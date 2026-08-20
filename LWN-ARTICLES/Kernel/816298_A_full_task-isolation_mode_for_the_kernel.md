---
title: "A full task-isolation mode for the kernel"
url: https://lwn.net/Articles/816298/
date: "April 6, 2020"
category: "Scheduler-CPU isolation"
author: "April 6, 2020 This article was contributed by Marta Rybczyńska"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

April 6, 2020

This article was contributed by Marta Rybczyńska

Some applications require guaranteed access to the CPU without even brief interruptions; realtime systems and high-bandwidth networking applications with user-space drivers can fall into the category. While Linux provides some support for CPU isolation (moving everything but the critical task off of one or more CPUs) now, it is an imperfect solution that is still subject to some interruptions. Work has been continuing in the community to improve the kernel's CPU-isolation capabilities, notably with improvements in the nohz (tickless) mode, but it is not finished yet. Recently, Alex Belits [submitted a patch set](<https://lwn.net/ml/linux-kernel/aed12dd15ea2981bc9554cfa8b5e273c1342c756.camel@marvell.com/>) (based on [work by Chris Metcalf in 2015](<https://lwn.net/Articles/659490/>)) that introduces a completely predictable environment for Linux applications -- as long as they do not need any kernel services.

#### Nohz and task isolation

Currently, the [nohz mode](<https://lwn.net/Articles/549580/>) in Linux allows partial task isolation. It decreases the number of interrupts that the CPU receives; for example, the clock tick interrupt is disabled for nearly all CPUs. However, nohz does not guarantee there will be no interruptions; the running task can still be interrupted by page faults (careful design of an application can avoid that) or delayed workqueues. The advantage of this mode is that the tasks can run regular code, including system calls. In addition to that, any additional overhead is limited to the system-call entry and exit paths.

For some applications, the lack of absolute guarantees from nohz may cause problems. As an example, high-performance, user-space network drivers that have a [small number of CPU cycles in which to handle each packet](<https://lwn.net/Articles/629155/>); for those, interrupt and interrupt handling may cause a significant delay in their response and use up to the entire time available. Realtime operating systems (RTOSes) can provide the needed guarantees, but they have limited hardware support; the authors of the patch feel that it is less work to develop and maintain interrupt-free applications than to support a RTOS next to Linux, as Belits [explained](<https://lwn.net/ml/linux-kernel/4473787e1b6bc3cc226067e8d122092a678b63de.camel@marvell.com/>):

The alternative, running RTOS instead of Linux, is becoming more and more labor-consuming because modern CPUs and SoCs have very complex device/resource configuration and management procedures, and at this point for some hardware it is clearly in the realm of impractical to maintain an RTOS with hardware support on par with Linux kernel, reliable and secure at the same time. 

In these times, even embedded systems often contain a number of cores, and system designers are adding more for tasks requiring predictability. Belits explained that further:

Therefore OS ability to switch a CPU core into RTOS-ish mode [...] is an important feature for modern embedded systems development. Probably more important than even real-time interrupts latency and preemption, now that people, when they don't like how their interrupts are handled, can just add CPU cores. 

The kernel currently has a couple of features meant to make it possible to run applications without interruptions: nohz (described above) and CPU isolation (or "isolcpus"). The latter feature isolates one or more CPUs — making them unavailable to the scheduler and only accessible to a process via an explicitly set affinity — so that any processes running there need not compete with the rest of the workload for CPU time. These features reduce interruptions on the isolated CPUs, but do not fully eliminate them; task isolation is an attempt to finish the job by removing _all_ interruptions. A process that enters the isolation mode will be able to run in user space with no interference from the kernel or other processes.

#### Configuring and activating task isolation

The authors assume that isolation is not needed in kernel space or during the task's initialization phase. A task enters the isolation mode at some point in time and stays in this mode until it leaves the isolation on its own, performs some action that causes the isolation to be broken, or receives a signal that was directed to it.

The kernel needs to be compiled with the `CONFIG_TASK_ISOLATION` flag and then booted with the same options as for nohz mode with CPU isolation: 

```
isolcpus=nohz,domain,CPULIST
```

where `nohz` disables the timer tick on the specified CPUs, `domain` removes the CPUs from the scheduling algorithms, and `CPULIST` is the list of CPUs where the isolation options are applied. Optionally, the `task_isolation_debug` kernel command-line option causes a stack backtrace when a task loses isolation.

When a task has finished its initialization, it can activate isolation by using the `PR_TASK_ISOLATION` operation provided by the `[prctl()](<http://man7.org/linux/man-pages/man2/prctl.2.html>)` system call. This operation may fail for either permanent or temporary reasons. An example of a permanent error is when the task is set up on a CPU without isolation; in this case, entering isolation mode is not possible. Temporary errors are indicated by the `EAGAIN` error code; examples include a time when the delayed workqueues could not be stopped. In such cases, the task may retry the operation if it wants to enter isolation, as it may succeed the next time.

In the `prctl()` call, the developer may also configure the signal to be sent to the task when it loses isolation. The additional macro to use is `PR_TASK_ISOLATION_SET_SIG()`, passing it the signal to send. The command then becomes similar to the one in the [example code](<https://github.com/abelits/libtmc/blob/feb17688baa9680fad3cdb2dff16c0a47d86f6b4/isol.c#L896>):

```
prctl(PR_SET_TASK_ISOLATION, PR_TASK_ISOLATION_ENABLE
              | PR_TASK_ISOLATION_SET_SIG(SIGUSR1), 0, 0, 0);
```

Here, the process has requested the receipt of a `SIGUSR1` signal rather than the default `SIGKILL` should it lose isolation. 

#### Losing isolation

The task will lose isolation if it enters kernel space as the result of a system call, a page fault, an exception, or an interrupt. The (fatal by default) signal will be sent when this happens, with a couple of exceptions: a `prctl()` call to turn off isolation, or `exit()` and `exit_group()`; these calls cause the task to exit, so the isolation mode is finished at that point.

When the task loses isolation by any means other than the above system calls, it will receive a signal, `SIGKILL` by default, which causes termination of the task. The signal can be modified, in the case the application prefers to catch it. This can be used, for example, if an application wants to log the information about lost isolation before exiting or attempt to rerun the code without isolation guarantees.

The task can enter and exit isolation when it desires. To leave isolation without a signal it should call:

```
prctl(PR_SET_TASK_ISOLATION, 0, 0, 0, 0);
```

#### The internals

When a process calls `prtcl()` to enable task isolation, it is marked with the `TIF_TASK_ISOLATION` flag in the kernel. The main part of the job of setting up task isolation, though, is done when returning from the `prctl()`. When the kernel returns to user space and sees the `TIF_TASK_ISOLATION` flag set, it arranges for the task not to be interrupted in the future. Interrupts are disabled, and the kernel disables any events that may interrupt the isolated CPU(s). In current patches, it disables the scheduler's clock tick and vmstat delayed work, and drains pages out of the per-CPU pagevec to avoid inter-processor interrupts (IPIs) for cache flushes. More isolation actions may be added in the future.

This isolation work is more straightforward in the current version than it was in the 2015 patch set. Since then, Linux has gained the ability to offload timer ticks from the isolated CPUs to so-called "housekeeping" CPUs -- all that are not on the CPU list of the `isolcpus` kernel option. That removes the need to make additional requirements for dealing with pending timers on CPUs before they can be isolated. 

The patch set also adds diagnostics on the non-isolated CPUs. If the kernel finds itself about to interrupt an isolated CPU, it will generate diagnostics (a warning in the kernel log by default, but a stack dump is also possible) on the interrupting CPU. Examples of such situations include sending an IPI or TLB flush. If an interrupt is not handled by Linux, for example a hypervisor interrupt, it can end up sending a reschedule IPI to an isolated CPU, causing the signal to notify the isolated task to be generated. With regard to that problem, Frédéric Weisbecker [wondered](<https://lwn.net/ml/linux-kernel/20200305183313.GA29033@lenoir/>) if support for hypervisors is even necessary, but no conclusion has been reached on this topic.

The task-isolation mode requires changes in the architecture code; the patch set includes implementations for x86, arm, and arm64. An architecture needs to define `HAVE_ARCH_TASK_ISOLATION` and the new `TIF_TASK_ISOLATION` task flag. It needs to change its interrupt and page-fault entry routines to add a call to `task_isolation_interrupt()` so that any isolated tasks will exit isolation. The reschedule IPI should call `task_isolation_remote()` for the same purpose. Finally the system-call code should invoke `task_isolation_syscall()` to check if the call is allowed. When exiting to user space it should call `task_isolation_check_run_cleanup()` to run pending cleanup and `task_isolation_start()` if the isolation flag is set for the current task.

Apart from the changes in the architecture-specific code, adding the isolation feature caused several changes in other kernel subsystems. For example, in the network code, `flush_all_backlogs()` will enqueue work only on non-isolated CPUs. The trace ring buffer behaves on isolated CPUs in a similar way to offline ones -- any updates will be done when the task exits isolation. Another change in the isolation mode is that kernel jobs are scheduled on housekeeping CPUs only. This includes tasks like probing for PCIe devices. Finally, `kick_all_cpus_sync()` has been modified to avoid scheduling interrupts on CPUs with isolated tasks. Weisbecker [did not agree](<https://lwn.net/ml/linux-kernel/20200306153446.GC8590@lenoir/>) with this approach and [listed a number of race conditions](<https://lwn.net/ml/linux-kernel/20200309022829.GB9615@lenoir/>) that may happen between this function and the task entering isolation. He suggested fixing the callers instead.

#### Summary

The patch set has received initial favorable reviews and it seems that this feature is of interest to developers. There are still some unresolved comments to be addressed and some patches did not receive a review yet. The patch set changes some basic kernel functions in a subtle way, so there will surely be questions asked about testing of the feature. In addition, of course, to the possible regressions. When those issues are resolved, it will likely be included in one of the upcoming kernel releases.
