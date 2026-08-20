---
title: "Various scheduler-related topics"
url: https://lwn.net/Articles/352863/
date: "September 15, 2009"
category: Latency; Scheduler
author: "By Jonathan Corbet September 15, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 15, 2009

Scheduler-related development seems to come in bursts. Things will be relatively quiet for a few development cycles, then activity will suddenly increase. We would appear to be in one of those periods where developers start to show a higher level of interest in what the scheduler is doing. The posting of the BFS scheduler has certainly motivated some of this activity, but there is more than that going on. 

#### Interactivity

On the BFS front, the (mildly) inflammatory part of the discussion would appear to have run its course. Anybody who has watched the linux-kernel list knows that serious attempts to fix problems often follow the storm; that appears to be the case this time around. Benchmarks are being posted by a number of people; as a general rule, the results of these benchmark runs tend to be mixed. There are also developers and users posting about problems that they are observing; see, for example, [Jens Axboe's report](<https://lwn.net/Articles/352875/>) of a ten-second pause while trying to run the `xmodmap` command. 

As part of the process of tracking down problems, the conversation turned to tuning the scheduler. Ingo Molnar [pointed out](<https://lwn.net/Articles/352880/>) that there is a whole set of flags governing scheduler behavior, all of which can be tweaked by the system administrator: 

Note, these flags are all runtime, the new settings take effect almost immediately (and at the latest it takes effect when a task has started up) and safe to do runtime. It basically gives us 32768 pluggable schedulers each with a slightly separate algorithm \- each setting in essence creates a new scheduler. 

The idea here is not that each user should be required to pick out the correct scheduler from a set of 32768 - a number which presumably seems high even to the "Linux is about choice" crowd. But these flags can be useful for anybody who is trying to track down why the behavior of the scheduler is not as good as it should be. When a tuning change improves things, it gives developers a hint about where they should be looking to find the source of the problem. 

A particular test suggested by Ingo was this: 

```
echo NO_NEW_FAIR_SLEEPERS > /debug/sched_features
```

(Politically-correct developers will, of course, have debugfs mounted under `/sys/kernel/debug`. Your editor takes no position on the proper debugfs mount point.) 

One tester [reported immediately](<https://lwn.net/Articles/352889/>) that setting this flag made the problems go away. Jens also noted that his ten-second `xmodmap` problem was solved. The evidence of problems with the NEW_FAIR_SLEEPERS feature was compelling enough that Ingo posted [a patch to disable it by default](<https://lwn.net/Articles/352890/>); that patch has been merged for 2.6.32. 

For the curious, the NEW_FAIR_SLEEPERS feature is a simple tweak which gives a process a small runtime credit when it returns to the run queue after a sleep. It is meant to help interactive processes, but, clearly, something is not working as expected. Once the real problem has been tracked down, it's possible that the NEW_FAIR_SLEEPERS feature could, once again, be enabled by default. In the mean time, users experiencing interactivity problems may want to try disabling it and seeing if things get better. 

#### Child-runs-first

Another default parameter is changing for 2.6.32; it controls which process runs first after a `fork()`. For much of the recent past, `fork()` has arranged things such that the child process gets to run before `fork()` returns to the parent; this behavior was based on the general observation that the child's work is often more important. There is a good reason to run the parent first, though: the parent's state is active in the processor, the translation lookaside buffer (TLB) contains the right information, etc. So parent-runs-first should perform better. It appears that recent tests showed that parent-runs-first does, indeed, outperform child-runs-first on that most important benchmark: kernel builds. That was enough to get the default changed. 

There are some concerns that this change could expose application bugs. Jesper Juhl [expresses those concerns](<https://lwn.net/Articles/352906/>) this way: 

I'm just worried that userspace programs have come to rely on a certain behaviour and changing that behaviour may result in undesired results for some apps. In a perfect world people would just fix those apps that (incorrectly) relied on a certain child-/parent-runs-first behaviour, but the world is not perfect, and many apps may not even have source available. 

Child-runs-first has never been a part of the `fork()` API, though; it's not something that applications should rely on. Even before the change, behavior could differ as a result of preemption, SMP systems, and more. So it's really true that child-runs-first was never guaranteed. But that will not make users feel any better if applications break. To help those users, there is a new `kernel.sched_child_runs_first` sysctl knob; setting it to one will restore the previous behavior. 

#### Better cpuidle governance

Active CPU scheduling is interesting, but there is also work happening in another area: what happens when nobody wants the CPU? Contemporary processors include a number of power management features which can be used to reduce power consumption when nothing is going on. Clearly, anybody who is concerned about power consumption will want the processor to be in a low-power state whenever possible. There are, however, some problems with a naive "go into a low power state when idle" policy: 

  * Transitions between power states will, themselves, consume power. If a CPU is put into a very low-power state, only to be brought back into operation a few microseconds later, the total power consumption will increase. 

  * Power state transitions have a performance cost. An extreme example would be simply pulling the plug altogether; power consumption will be admirably low, but the system will experience poor response times that not even the BFS scheduler can fix. Putting the CPU into a more conventional low-power state will still create latencies; it takes a while for the processor to get back into a working mode. So going into a low-power state too easily will hurt the performance of the system. 

It turns out that the CPU "governor" code in the mainline kernel often gets this decision wrong, especially for the newer Intel "Nehalem" processors; the result is wasted energy and poor performance, where "poor performance" means a nearly 50% hit on some tests that Arjan van de Ven ran. His response was to put together [a patch](<http://lwn.net/Articles/352180/>) aimed at fixing the problems. The approach taken is interesting. 

Clearly, it makes no sense to put the processor into a low-power state if it will be brought back to full power in the very near future. So all the governor code really has to do is to come up with a convincing prediction of the future so it knows when the CPU will be needed again. Unfortunately, the chip vendors have delayed the availability of the long-promised crystal-ball peripherals yet again, forcing the governor code to rely on heuristics; once again, software must make up for deficiencies in the hardware. 

When trying to make a guess about when a CPU might wake up, there are two things to consider. One is entirely well known: the time of the next scheduled timer event. The timer will put an upper bound on the time that the CPU might sleep, but it is not a definitive number; interrupts may wake up the CPU before the timer goes off. Arjan's governor tries to guess when that interrupt might happen by looking at the previous behavior of the system. Every time that the processor wakes up, the governor code calculates the difference between the estimated and actual idle times. A running average of that difference is maintained and used to make a (hopefully) more accurate guess as to what the next idle time will really be. 

Actually, several running averages are kept. The probability of a very long idle stretch being interrupted by an interrupt is rather higher than the probability when expected idle period is quite short. So there is a separate correction factor maintained for each order of magnitude of idle time - a 1ms estimate will have a different correction factor than a 100µs or a 10ms guess will. Beyond that, a completely different set of correction factors is used (and maintained) if there is I/O outstanding on the current CPU. If there are processes waiting on short-term (block) I/O, the chances of an early wakeup are higher. 

The performance concern, meanwhile, is addressed by trying to come up with some sort of estimate of how badly power-management latency would hurt the system. A CPU which is doing very little work will probably cause little pain if it goes to sleep for a while. If, instead, the CPU is quite busy, it's probably better to stay powered up and ready to work. In an attempt to quantify "busy," the governor code calculates a "multiplier": 

```
multiplier = 1 + 20*load_average + 10*iowait_count
```

All of the numbers are specific to the current CPU. So the multiplier is heavily influenced by the system load average, and a bit less so by the number of processes waiting for I/O. Or so it seems - but remember that processes in uninterruptible waits (as are used for block I/O) are counted in the load average, so their influence is higher than it might seem. In summary, this multiplier grows quickly as the number of active processes increases. 

The final step is to examine all of the possible sleep states that the processor provides, starting with the deepest sleep. Each sleep state has an associated "exit latency" value, describing how long it takes to get out of that state; deeper sleeps have higher exit latencies. The new governor code multiplies the exit latency by the multiplier calculated above, then compares the result to its best guess for the idle time. If that idle time exceeds the adjusted latency value, that sleep state is chosen. Given the large multipliers involved, one can see that expected idle times must get fairly long fairly quickly as the system load goes up. 

According to Arjan, this change restores performance to something very close to that of a system which is not using sleep states at all. The improvement is significant enough that Arjan would like to see the code merged for 2.6.32, even though it just appeared during the merge window. That might happen, though it is possible that it will turned into a separate CPU governor for one development cycle just in case regressions turn up.
