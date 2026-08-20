---
title: CPU frequency governors and remote callbacks
url: https://lwn.net/Articles/732740/
date: "September 4, 2017"
category: "Power management-Frequency scaling; cpufreq"
author: "September 4, 2017 This article was contributed by Viresh Kumar"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

September 4, 2017

This article was contributed by [Viresh Kumar](<mailto:viresh.kumar@linaro.org>)

The kernel's CPU-frequency ("cpufreq") governors are charged with picking an operating frequency for each processor that minimizes power use while maintaining an adequate level of performance as determined by the current policy. These governors normally run locally, with each CPU handling its own frequency management. The 4.14 kernel release, though, will enable the CPU-frequency governors to control the frequency of any CPU in the system if the architecture permits, a change that should improve the performance of the system overall. 

For a long time, the cpufreq governors used the kernel's timer infrastructure to run at a regular interval and sample CPU utilization. That approach had its shortcomings; the biggest one was that the cpufreq governors were running in a reactive mode, choosing the next frequency based on the load pattern in the previous sampling period. There is, of course, no guarantee that the same load pattern will continue after the frequency is changed. Additionally, there was no coordination between the cpufreq governors and the task scheduler. It would be far better if the cpufreq governors were proactive and, working with the scheduler, could choose a frequency that suits the load that the system is going to have in the next sampling period.

In the 4.6 development cycle, Rafael Wysocki removed the dependency on kernel timers and placed hooks within the scheduler itself. The scheduler calls these hooks for certain events, such as attaching a task to a run queue or when the load created by the processes in run queue changes. The hooks are implemented by the individual cpufreq governors. Those governors register and unregister their CPU-utilization update callbacks with the scheduler using the following interfaces:

```
void cpufreq_add_update_util_hook(int cpu, struct update_util_data *data,
                            	      void (*func)(struct update_util_data *data, 
    						   u64 time, unsigned int flags));
        void cpufreq_remove_update_util_hook(int cpu);
```

Where `struct update_util_data` is defined as: 

```
struct update_util_data {
    	void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
        };
```

The scheduler internally keeps per-CPU pointers to the `struct update_util_data` which is passed to the `cpufreq_add_update_util_hook()` routine. Only one callback can be registered per CPU. The scheduler starts calling the `cpufreq_update_util_data->func()` callback from the next event that happens after the callback is registered. 

The legacy governors (ondemand and conservative) are still considered to be reactive, as they continue to rely on the data available from the last sampling period to compute the next frequency to run. Specifically, they calculate CPU load based on how much time a CPU was idle in the last sampling period. However, the [schedutil governor](<https://lwn.net/Articles/682391/>) is considered to be proactive, since it calculates the next frequency based on the average utilization of the CPU's current run queue. The schedutil governor will pick the maximum frequency for a CPU if any realtime or deadline tasks are available to run.

#### Remote callbacks

In current kernels, the scheduler will call these utilization-update hooks only if the target run queue, the queue for the CPU whose utilization has changed, is the run queue of the local CPU. While this works well for most scheduler events, it doesn't work that well for some. This mostly affects performance of only the schedutil cpufreq governor, since the others don't take the average utilization into consideration when calculating the next frequency.

With certain types of systems, such as Android, the latency of cpufreq response to certain scheduling events can be critical. As the cpufreq callbacks aren't called from remote CPUs currently, it means there are certain situations where a target CPU may not run the cpufreq governor for some time. 

For example, consider a system where a task is running on a given CPU, and a second task is queued to run on that CPU by a different CPU. If the newly enqueued task has a high CPU demand, the target CPU should increase its frequency immediately (based on the utilization average of its run queue) to meet that demand. But, because of the above-mentioned limitation, this does not occur as the task was enqueued by a remote CPU. The schedutil cpufreq governor's utilization update hook will be called only on the next scheduler event, which may happen only after some microseconds have passed. That is bad for performance-critical tasks like the Android user interface. Most Android devices refresh the screen at 60 frames per second; that is 16ms per frame. The screen rendering has to finish within these 16ms to avoid jerky motion. If 4ms are taken by the cpufreq governor to update the frequency, then the user's experience isn't going to be nice.

This problem can be avoided by invoking the governor to change the target CPU's frequency immediately after queuing the new task, but that may not always be possible or practical; the processor architecture may not allow it. For example, the x86 architecture updates CPU frequencies by writing to local, per-CPU registers, which remote CPUs cannot do. Sending an inter-processor interrupt to the target CPU to update its frequency sounds like overkill and will add unnecessary noise for the scheduler. Using interrupts could add just the sort of latency that this work seeks to avoid. 

On the other hand, updating CPU frequencies on the ARM architecture is normally CPU-independent; any CPU can change the frequency of any other CPU. Thus, the [patch set](<https://marc.info/?l=linux-kernel&m=150122447311329&w=2>) enabling remote callbacks took the middle approach and avoided sending inter-processor interrupts to the target CPU. The patch set is queued in the power-management tree for the 4.14-rc1 kernel release. The frequency of a CPU can now be changed remotely by a CPU that shares cpufreq policy with the target CPU; that is, both the CPUs share their clock and voltage rails and switch performance state together. But CPU-frequency changes can also be made from any other CPU on the system if the cpufreq policy of the target CPU has the `policy->dvfs_possible_from_any_cpu` field set to true. This is a new field and must be set by the cpufreq driver from its `cpufreq_driver->init()` callback if it allows changing frequencies from CPUs running a different cpufreq policy. The generic device-tree based cpufreq driver is already updated to enable remote changes. 

Remote cpufreq callbacks will be enabled (by default) in the 4.14 kernel release; they should improve the performance of the schedutil governor in a number of scenarios. Other architectures may want to consider updating their cpufreq drivers to set `policy->dvfs_possible_from_any_cpu` field to true if they can support cross-CPU frequency changes.
