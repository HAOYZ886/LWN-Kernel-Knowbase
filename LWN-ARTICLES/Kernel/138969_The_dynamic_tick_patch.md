---
title: The dynamic tick patch
url: https://lwn.net/Articles/138969/
date: "June 7, 2005"
category: Interrupts; Timer frequency
author: 
---

The timer interrupt is one of the most predictable events on a Linux system. Like a heartbeat, it pokes the kernel every so often (about every 1ms on most systems), enabling the kernel to note the passage of time, run internal timers, etc. Most of the time, the timer interrupt handler just does its job and nobody really notices. 

There are times, however, when this interrupt can be unwelcome. Many processors, when idle, can go into a low-power state until some work comes along. To such processors, the timer interrupt looks like work. If there is nothing which actually needs to be done, however, then the processor might be powering up 1000 times per second for no real purpose. Timer interrupts can also be an issue on virtualized systems; if a system is hosting dozens of Linux instances simultaneously, the combined load from each instance's timer interrupt can add up to a substantial amount of work. So it has often been thought that there would be a benefit to turning off the timer interrupt when there is nothing for the system to do. 

Tony Lindgren's [dynamic tick patch](<https://lwn.net/Articles/138264/>) is another attempt to put a lid on the timer interrupt. This version of the patch only works on the i386 architecture, but it is simple enough that porting it to other platforms should not be particularly difficult. 

The core of the patch is a hook into the architecture-specific `cpu_idle()` function. If a processor has run out of work and is about to go idle, it first makes a call to `dyn_tick_reprogram_timer()`. That function checks to see whether all other processors on the system are idle; if at least one processor remains busy, the timer interrupt continues as always. Experience has shown that trying to play games with the timer interrupt while the system is loaded leads to a net loss in performance - the overhead of reprogramming the clock outweighs the savings. So, if the system is working, no changes are made to the timer. 

If, instead, all CPUs on the system are idle, there may be an opportunity to shut down the timer interrupt for a while. When the system goes idle, there are only two events which can create new work to do: the completion of an I/O operation or the expiration of an internal kernel timer. The dynamic tick code looks at when the next internal timer is set to go off, and figures it might be able to get away with turning off the hardware timer interrupt until then. After applying some tests (there are minimum and maximum allowable numbers of interrupts to skip), the code reprograms the hardware clock to interrupt after this time period, and puts the processor to sleep. 

At some point in the future, an interrupt will come along and wake the processor. It might be the clock interrupt which had been requested before, or it could be some other device - a keyboard or network interface, for example. The dynamic tick code hooks into the main interrupt handler, causing its own handler to be invoked for every interrupt on the system, regardless of source. This code will figure out how many clock interrupts were actually skipped, then loop calling `do_timer_interrupt()` until it catches up with the current time. Finally, the interrupt handler restores the regular timer interrupt, and the system continues as usual. 

The end result is a system which can drop down to about 6 timer interrupts per second when nothing is going on. That should eventually translate into welcome news for laptop users and virtual hosters running Linux.
