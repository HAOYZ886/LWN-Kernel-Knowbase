---
title: Teaching the scheduler about power management
url: https://lwn.net/Articles/602479/
date: "June 18, 2014"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "June 18, 2014 This article was contributed by Nicolas Pitre"
---

June 18, 2014

This article was contributed by Nicolas Pitre

Power-efficient CPU scheduling is increasingly important in the mobile world, but it has become just as important in large data-center settings, where the electricity bills can be painful indeed. Unfortunately, the kernel's infrastructure for CPU power management lacks integration with the scheduler itself, with the result that scheduling decisions are not as good as they should be. This article reviews the state of the CPU power-management mechanisms in the kernel and looks at what is being done to improve the situation. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### A bit of history

A process scheduler is the core component of an operating system responsible for selecting which process to run next. The scheduler implementation in the Linux kernel has been through a couple of iterations — and even complete rewrites — over the years. The [Completely Fair Scheduler (CFS)](<http://en.wikipedia.org/wiki/Completely_Fair_Scheduler>), written by Ingo Molnar, was [introduced](<http://lwn.net/Articles/230501/>) in the 2.6.23 kernel. It replaced the [O(1) scheduler](<http://en.wikipedia.org/wiki/O\(1\)_scheduler>) which, in turn, was [introduced](<http://lwn.net/2002/0110/a/scheduler-e1.php3>) in version 2.5.2 of the kernel, also by Ingo, replacing the scheduler implementation that existed before that. Despite all of the different algorithms, the general goal is always the same: to try to make the most of available CPU resources.

CPU resources have also evolved during this time. Initially, the scheduler's role was simply to properly manage processor time between all runnable processes. Increasing parallelism in hardware due to the emergence of [SMP](<http://en.wikipedia.org/wiki/Symmetric_multiprocessing>), [SMT](<http://en.wikipedia.org/wiki/Simultaneous_multithreading>) (or Hyper-threading), and [NUMA](<http://en.wikipedia.org/wiki/Non-uniform_memory_access>) added more twists to the problem. And, of course, the scheduler had to scale to an ever-increasing number of processes and processors in the same system without consuming too much CPU time on its own. These changes explain why multiple scheduler implementations have been developed over the last half-century and are still being worked on today. In the process, the scheduler has grown in complexity and only a few individuals have become experts in this area.

Initially, task scheduling was all about throughput with no regard for energy consumption; scheduler work was driven by the enterprise space, where everything was plugged into the wall. At the other end of the spectrum, we saw the emergence of battery-operated devices from the embedded and mobile space, where power management is a primary concern. Separate subsystems dealing with power management, such as [cpuidle](<http://lwn.net/Articles/384146/>) and [cpufreq](<https://www.kernel.org/doc/Documentation/cpu-freq/governors.txt>), were introduced and contributed to by a different set of developers with little scheduler experience. In due course, the power management subsystems grew in complexity as well with its own experts.

This split arrangement worked out reasonably well… at least initially. The isolation between the subsystems allowed for easier development and maintenance. With mobile devices growing in capabilities, as well as ever-increasing data-center electric bills, everyone started caring about energy efficiency. This brought about core kernel changes such as [deferrable timers](<http://lwn.net/Articles/228143/>), [dyntick](<http://lwn.net/Articles/223185/>), and [runtime power management](<http://lwn.net/Articles/347573/>). The rise of multi-core portable devices pushed the need for yet more aggressive power-management tricks such as the [controversial use](<http://lwn.net/Articles/570392/>) of CPU offlining.

There is a pattern that emerges from these changes: the more complex scheduler and power management become, the more isolated they are from each other. And this turns out to be completely counterproductive, since, as we'll see later, one side can't predict what the other side might do in the near future. Because (or in spite) of that, some chip manufacturers are increasingly implementing [DVFS](<http://en.wikipedia.org/wiki/Voltage_and_frequency_scaling>) in hardware away from the operating system, which exacerbates the problem. Yet [support for ARM's big.LITTLE](<http://lwn.net/Articles/481055/>) and the increasing influence scheduler decisions have on power consumption in general have [made it clear](<http://lwn.net/Articles/552889/>) that merging power management with the scheduler is becoming unavoidable. 

#### Scheduler: meet cpuidle

The cpuidle subsystem tries to minimize power consumption by selecting a low-power mode, or idle mode (often referred as [C-State](<http://www.hardwaresecrets.com/article/611>)), when the CPU is idle. However, idling a CPU comes with a price: the more power savings such a mode provides, the longer it will take for the affected CPU to become fully operational again. A good balance between the power actually saved and the time "wasted" in entering and exiting a power-saving mode has to be reached. Furthermore, many modes consume some non-negligible amount of power for the CPU simply to transition in and out of them, meaning the CPU has to be idle for a sufficiently long period of time for those modes to be worth entering. Most CPUs have multiple idle modes, providing different trade-offs between achievable power savings and latency.

Therefore, the cpuidle code has to gather statistics on actual CPU usage to select the most appropriate idle mode depending on the observed idleness pattern of the CPU. And this statistics-gathering work duplicates what the scheduler already does, albeit through indirect and somewhat imprecise heuristics.

Idleness patterns are determined by wake-up events bringing the CPU out of idle. Those events can be classified into three categories:

  * **Predictable events** : This group comprises all timers from which we can obtain their next expiry time and deduce a minimum idle period. 

  * **Semi-predictable events** : These are somewhat repetitive events, like I/O request completions, that often follow a known pattern. 

  * **Random events** : Anything else, such as keystrokes, touchscreen events, network packets, etc. 

By directly involving the scheduler in the idle-state selection process, we can do a much better job at considering the semi-predictable events. I/O patterns are mainly a function of those tasks generating them and the device they're directed to. The scheduler can therefore keep track of the average I/O latency on a per-task basis and, possibly, with some input from the [I/O scheduler](<http://en.wikipedia.org/wiki/I/O_scheduling>), provide an estimated delay for the next I/O completion to occur according to the list of waiting tasks on a given CPU. And if a task is migrated to a different CPU, its I/O latency statistics are migrated along. The scheduler is therefore in a better position to appreciate the actual idleness of a CPU.

It is therefore necessary for the scheduler and cpuidle to become better integrated, to let the scheduler manage the available idle modes and eventually bypass the current cpuidle governors entirely. Moving the main idle loop into the scheduler code will also allow for better accounting of CPU time spent in the servicing of interrupts and their occurrence rate while idle.

Furthermore, the scheduler should be aware of the current idle mode on each CPU to do a better job at load balancing. For instance, let's consider the function `find_idlest_cpu()` in `kernel/sched/fair.c`, which looks for the least-loaded CPU by comparing the weighted CPU load value for each CPU. If multiple CPUs are completely idle, their load would be zero, with no distinction for the idle mode they're in. In this case, it would be highly beneficial to choose the CPU whose current idle mode has the shortest exit latency. If idle exit latency is the same for all idle CPU candidates then the last to have entered idle mode is more likely to have a warmer cache (assuming the relevant idle mode preserves cache, of course). An [initial patch series](<http://lkml.org/lkml/2014/3/28/181>) to that effect was posted by Daniel Lezcano.

This also [highlighted](<http://lkml.org/lkml/2014/4/15/364>) the fact that some definitions for the same expression may differ depending on one's perspective. A function called `find_idlest_cpu()` in the scheduler context is simply the converse of `find_busiest_cpu()`, whereas in the cpuidle context this would mean looking for the CPU with the deepest idle state. The deeper an idle state is, the more costly it is to bring a CPU back to operational state -- clearly not what we want here. A similar confusion may occur with the word "power". The traditional meaning in the scheduler code is "compute capacity" while it means "energy consumption rate" in a power management context. [Patches](<http://lkml.org/lkml/2014/5/26/614>) to clarify this have recently been merged.

#### Scheduler: meet cpufreq

The scheduler keeps track of the average amount of work being done by scheduled tasks on each CPU in order to give each task fair access to CPU resources and to decide when to perform load balancing. The [ondemand cpufreq governor](<http://lwn.net/Articles/384132/>) does similar load tracking in order to dynamically set each CPU's clock frequency to optimize battery life. Since energy consumption is proportional to the square of the voltage, it is desirable to run at the lowest clock frequency, which allows for voltage reduction to the CPU, while still being fast enough to perform all the scheduled work during a given period of time. 

As with cpuidle, the cpufreq subsystem was developed in isolation from the CPU scheduler. Many problems result from the split between those subsystems:

  * The cpufreq code goes to great lengths trying to evaluate the actual CPU load through indirect means, including heuristics to avoid misprediction, while, once again, the scheduler has all this information available already. 

  * The scheduler can determine the [load contribution of individual tasks](<http://lwn.net/Articles/531853/>), whereas the cpufreq code has no such ability. In the occurrence of a task migration, or a task waking up, the scheduler may determine in advance what the load on the target CPU is likely to become. The cpufreq code may only notice an average load increase and react to it after the fact. 

  * The scheduler records the execution time for each task in order to ensure fairness between all tasks. However, since the scheduler has no awareness of CPU frequency changes, tasks executing on a CPU whose clock has been slowed down will be unfairly charged more execution time than similar tasks running on another CPU with a faster clock for the same amount of work. Fairness is thus compromised. 

  * As the CPU clock frequency is reduced, the resulting apparent increase in task load may trigger load balancing toward a less-loaded CPU in order to spread the load, despite the fact that this increased apparent load was indeed the cpufreq's goal initially. 

  * And if that load balancing happens while the target CPU's clock frequency is reduced, then that CPU could end up being oversubscribed. Because there is no coordination between the scheduler and cpufreq, either (or both) of them may react by, respectively, migrating a task back or raising the CPU clock frequency. The CPU may suddenly be underutilized, and the cycle could repeat again. 

To fix this, the current plan is to integrate cpufreq more tightly with the scheduler. The various platform-specific, low-level cpufreq drivers will remain unchanged and still register with the cpufreq core as usual; however, the governors — the part that decides what clock frequency to ask for and when — could be bypassed entirely. In fact, the scheduler could simply register itself as a new governor with the cpufreq core.

The advantage of a tighter integration of cpufreq with the scheduler is the ability to be proactive with clock frequency changes rather than reactive, and also to coordinate better with scheduler activities like load balancing. A CPU clock frequency change could be requested in anticipation of a load change; this could happen in response to a call to `fork()`, `exit()`, or when a task sleeps or wakes up. The frequency policy could be different depending on the particular scheduler event, task historical behavior patterns, etc.

However, to be able to perform well in the presence of varying CPU clock frequencies, the notion of scale-invariant task load tracking must be added to the scheduler. This is a correction factor to normalize load measurements from CPUs executing code at different speeds so the load contribution of a task can be predicted when the task is moved. The relative computation capacity of each CPU as seen by the scheduler also has to be adjusted according to its effective clock frequency in order to do proper load balancing. It is still unclear how accurate this correction factor can be, considering that tasks making lots of memory accesses are less likely to be influenced by the CPU clock speed compared to tasks performing computational work, etc. Still, anything is going to be better than no correction at all like we have today.

Incidentally, the scale-invariant load tracking does apply to big.LITTLE scheduling as well. Leaving cpufreq aside for a moment, a "little" CPU is permanently slowed down, which translates into a reduced compute capacity to the scheduler, and conversely a "big" CPU has more capacity. With distinct correction factors permanently applied to "big" and "little" CPUs, the scheduler is likely to just work optimally in terms of task throughput, with no further changes to the scheduler. The cpufreq correction factor simply has to be combined with the "big" and "little" factors afterward.

Several developers, including Mike Turquette and Tuukka Tikkanen, are working on the [cpufreq integration](<http://www.slideshare.net/linaroorg/lca14-306-cpuidlecpufreqintegrationwithscheduler>) and initial patches should be posted for public review soon.

#### Scheduler: may the power be with you

Okay… So we might get to the point where cpuidle and cpufreq are tightly integrated with the scheduler. Are we done? Unlikely. In fact we now have more difficult decisions to make than before and they all relate to the new mechanisms at the scheduler's disposal to perform load balancing. For example:

  * When the system load goes up, should a new CPU be brought out of idle, or should the clock frequency on an already running CPU be increased instead? Or both? 

  * Conversely, when the system load goes down, is it best to keep more CPUs alive with a reduced clock frequency or pack more tasks on fewer CPUs in order to send the other CPUs to sleep? 

  * Is it best to consolidate loads onto fewer CPUs or to spread the load over more CPUs? 

  * When is it time to perform active [task packing](<http://lwn.net/Articles/520857/>) to let a whole CPU cluster (or package) get into low-power mode? 

The latest [power-aware scheduler](<http://lwn.net/Articles/600419/>) work from Morten Rasmussen provides a framework to evaluate the power cost of the available scheduling scenarios. This, in combination with Vincent Guittot's [sched_domain topology](<https://lkml.org/lkml/2014/3/19/377>) and [CPU capacity tracking consolidation](<https://lkml.org/lkml/2014/5/23/458>) work, should provide answers to the above questions.

#### What else?

We desperately need measurement tools to validate proposed solutions. Linaro is working on a tool called [idlestat](<https://wiki.linaro.org/WorkingGroups/PowerManagement/Resources/Tools/Idlestat>) to validate idle-state usage and its effectiveness. Traditional benchmark tools such as [sysbench](<http://www.howtoforge.com/how-to-benchmark-your-system-cpu-file-io-mysql-with-sysbench>) may be combined with [energy usage monitoring](<http://lkml.org/lkml/2014/6/6/286>) to provide a way to perform power characterization of a system. Extensions to [cyclictest](<https://rt.wiki.kernel.org/index.php/Cyclictest>) to create various synthetic workloads are being explored as well. This is still unwieldy, though, and more integration and automation are required.

This article hasn't covered [thermal management](<http://en.wikipedia.org/wiki/Thermal_management_of_electronic_devices_and_systems>). The Linux kernel implements a thermal-management [interface](<https://www.kernel.org/doc/Documentation/thermal/sysfs-api.txt>) that allows a [user-space daemon](<http://www.linux.com/news/featured-blogs/200-libby-clark/721494-linux-thermal-daemon-monitors-and-controls-temperature-in-tablets-laptops>) to control thermal constraints. However, as we've seen, power-related issues are intertwined, and a thermal-control solution that lives separately from the scheduler is likely to be suboptimal. If the scheduler controls power states, it will also have to deal with platform temperature someday, providing [thermal "provisioning"](<http://lwn.net/Articles/599598/>) or the like. But we can save this for another day.

#### Thanks

Thanks to Amit Kucheria, Daniel Lezcano, Kevin Hilman, Mike Turquette, and Vincent Guittot for their help in reviewing this article.
