---
title: The tick broadcast framework
url: https://lwn.net/Articles/574962/
date: "November 26, 2013"
category: Timers
author: "November 26, 2013 This article was contributed by Preeti U Murthy."
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

November 26, 2013

This article was contributed by Preeti U Murthy.

Power management is an increasingly important responsibility of almost every subsystem in the Linux kernel. One of the most established power management mechanisms in the kernel is the [cpuidle framework](<https://lwn.net/Articles/384146/>) which puts idle CPUs into sleeping states until they have work to do. These sleeping states are called the "C-states" or CPU operating states. The deeper a C-state, the more power is conserved. 

However, an interesting problem surfaces when CPUs enter certain deep C-states. Idle CPUs are typically woken up by their respective local timers when there is work to be done, but what happens if these CPUs enter deep C-states in which these timers stop working? Who will wake up the CPUs in time to handle the work scheduled on them? This is where the "tick broadcast framework" steps in. It assigns a clock device that is not affected by the C-states of the CPUs as the timer responsible for handling the wakeup of all those CPUs that enter deep C-states. 

#### Overview of the tick broadcast framework

In the case of an idle or a semi-idle system, there could be more than one CPU entering a deep idle state where the local timer stops. These CPUs may have different wakeup times. How is it possible to keep track of when to wake up the CPUs, considering a timer is merely a clock device that cannot keep track of more information than the time at which it is supposed to interrupt? The tick broadcast framework in the kernel provides the necessary infrastructure to handle the wakeup of such CPUs at the right time. 

Before looking into the tick broadcast framework, it is important to understand how the CPUs themselves keep track locally of when their respective pending events need to be run. 

The kernel keeps track of the time at which a deferred task needs to be run based on the concept of timeouts. The timeouts are implemented using clock devices called timers which have the capacity to raise an interrupt at a specified time. In the kernel, such devices are called the "clock event" devices. Each CPU is equipped with a local clock event device that is programmed to interrupt at the time of the next-to-run deferred task on that CPU, so that said task can be scheduled on the CPU. These local clock event devices can also be programmed to fire periodically to do regular housekeeping jobs like updating the `jiffies` value, checking if a task has to be scheduled out, etc. These timers are therefore called the "tick devices" in the kernel and are represented by `struct tick_device`. 

A per-CPU `tick_device` representing the local timer is declared using the variable `tick_cpu_device`. Every CPU keeps track of when its local timer needs to interrupt it next in its copy of `tick_cpu_device` as `next_event` and programs the local timer with this value. To be more precise, the value can be found in `tick_cpu_device->evtdev->next_event`, where `evtdev` is an instance of the clock event device mentioned above. 

The external clock device that is required to stand in for the local timers in some deep idle states is just another tick device, but is not normally required to keep track of events for specific CPUs. This device is represented by `tick_broadcast_device` (defined in `kernel/time/tick-broadcast.c`), in contrast to `tick_cpu_device`. 

#### Registering a timer as the tick_broadcast_device

During the initialization of the kernel, every timer in the system registers itself as a `tick_device`. In the kernel, these timers are associated with some flags which define their properties. That property which is of special interest to us is represented by the flag `CLOCK_EVT_FEAT_C3STOP`. This means that in the C3 idle state, the timer stops. Although the C3 idle state is specific to the x86 architecture, this feature flag is generally used to convey that the timer stops in one of the deep idle states. 

Any timers which do not have the flag `CLOCK_EVT_FEAT_C3STOP` set are potential candidates for `tick_broadcast_device`. Since all local timers have this flag set on architectures where they stop in deep idle states, all of them become ineligible for this role. On architectures like x86, there is an external device called the HPET — High Precision Event Timer — which becomes a suitable candidate. Since the HPET is placed external to the processor, the idle power management for a CPU does not affect it. Naturally it does not have the `CLOCK_EVT_FEAT_C3STOP` flag set among its properties and becomes the choice for `tick_broadcast_device`. 

#### Tracking the CPUs in deep idle states

Now we'll return to the way the tick broadcast framework keeps track of when to wake up the CPUs that enter idle states when their local timers stop. Just before a CPU enters such an idle state, it calls into the tick broadcast framework. This CPU is then added to a list of CPUs to be woken up; specifically, a bit is set for this CPU in a "broadcast mask". 

Then a check is made to see if the time at which this CPU has to be woken up is prior to the time at which the `tick_broadcast_device` has been currently programmed. If so, the time at which the `tick_broadcast_device` should interrupt is updated to reflect the new value and this value is programmed into the external timer. The `tick_cpu_device` of the CPU that is going to deep idle state is now put in `CLOCK_EVT_MODE_SHUTDOWN` mode, meaning that it is no longer functional. 

Each time a CPU goes to deep idle state, the above steps are repeated and the `tick_broadcast_device` is programmed to fire at the earliest of the wakeup times of the CPUs in deep idle states. 

#### Waking up the CPUs in deep idle states

When the external timer expires, it interrupts one of the online CPUs, which scans the list of CPUs that have asked to be woken up to check if any of their wakeup times have been reached. That means the current time is compared to the `tick_cpu_device->evtdev->next_event` of each CPU. All those CPUs for which this is true are added to a temporary mask (different from the broadcast mask) and the appropriate next expiry time of the `tick_broadcast_device` is set to the earliest wakeup time of those CPUs. What remains to be seen is how the CPUs in the temporary mask are woken up. 

Every tick device has a "broadcast method" associated with it. This method is an architecture-specific function encapsulating the way inter-processor interrupts (IPIs) are sent to a group of CPUs. Similarly, each local timer is also associated with this method. The broadcast method of the local timer of one of the CPUs in the temporary mask is invoked by passing it the same mask. IPIs are then sent to all the CPUs that are present in this mask. Since wakeup interrupts are sent to a group of CPUs, this framework is called the "broadcast" framework. The broadcast is done in `tick_do_broadcast()` in `kernel/time/tick-broadcast.c`. 

The IPI handler for this specific interrupt needs to be that of the local timer interrupt itself so that the CPUs in deep idle states wake up as if they were interrupted by the local timers themselves. The effects of their local timers stopping on entering an idle state is hidden away from them; they should see the same state before and after wakeup and continue running like nothing had happened. 

While handling the IPI, the CPUs call into the tick broadcast framework so that they can be removed from the broadcast mask, since it is known that they have received the IPI and have woken up. Their respective tick devices are brought out of the `CLOCK_EVT_MODE_SHUTDOWN` mode, indicating that they are back to being functional. 

#### Conclusion

As can be seen from the above discussion, enabling deep idle states cause the kernel to have to do additional work. One would therefore naturally wonder if it is worth going through this trouble, since it could hamper performance in the process of saving power. 

Idle CPUs enter deep C-states only if they are predicted to remain idle for a long time, on the order of milliseconds. Therefore, broadcast IPIs should be well spaced in time and are not so frequent as to affect the performance of the system. We could further optimize the tick broadcast framework by aligning the wakeup time of the idle CPUs to a periodic tick boundary whose interval is of the order of a few milliseconds so that CPUs going to idle at almost the same time choose the same wakeup time. By looking at more such ways to minimize the number of broadcast IPIs sent we could ensure that the overhead involved is insignificant compared to the large power savings that the deep idle states yield us. If this can be achieved, it is a good enough reason to enable and optimize an infrastructure for the use of deep idle states. 

#### Acknowledgments

I would like to thank my technical mentor Vaidyanathan Srinivasan for having patiently reviewed the initial drafts, my manager Tarundeep Singh, and my teammates Srivatsa S. Bhat and Deepthi Dharwar for their guidance and encouragement during the drafting this article. 

Many thanks to IBM Linux Technology Center and LWN for having provided this opportunity.
