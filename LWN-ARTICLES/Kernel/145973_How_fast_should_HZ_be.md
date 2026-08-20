---
title: "How fast should HZ be?"
url: https://lwn.net/Articles/145973/
date: "August 2, 2005"
category: HZ; Timer frequency
author: 
---

There has been a debate slowly simmering on linux-kernel over an issue which, to most Linux users, will be invisible. Still, it points at the sorts of tradeoffs which must be made when configuring a system, and thus merits a look. 

One of the features which will be included in the 2.6.13 kernel is the ability to configure the frequency of the timer interrupt at kernel build time - at least, on the i386 architecture. This capability, by itself, is not controversial, but the new default value for HZ (250) is. Some developers think it is too low, while others (fewer) think it is too high. It does not appear that there is a single "right" value for this variable. 

HZ is the frequency with which the system's timer hardware is programmed to interrupt the kernel. Much of the kernel's internal housekeeping, including process accounting, scheduler time slice accounting, and internal time management, is done in the timer interrupt handler. Thus, the frequency of the timer interrupt affects a number of things; in particular, it puts an upper bound on the resolution of timers used with the kernel. If HZ is 1000 (the i386 default for 2.6 kernels through 2.6.12), then timers will have a best-case resolution of 1ms. If, instead, HZ is 100 (the 2.4 and prior default), that resolution is 10ms. 

The 250Hz default in 2.6.13 gives a maximum timer resolution of 4ms, which is said to be insufficient for many multimedia-oriented applications (and others which need higher-resolution timers). Such applications, in that environment, will be forced to use busy-waiting to achieve delays which are below the best resolution offered by the system, with the usual effect on CPU utilization. It is not the way the developers of these applications want to go. 

The arguments in favor of reducing HZ center around efficiency. A slower timer interrupt is said to require less power, since the processor (if relatively idle) will wake up less often. Thus, a lower value of HZ is supposed to be better for laptop users. The timer interrupt handler also requires CPU time (and a context switch, and cache space) every time it runs; running that handler less often will clearly reduce its overhead. 

Part of the problem, however, is that nobody has quantified the savings which can be expected from a slower timer interrupt. That changed, however, when Marc Ballarin posted [some results](<https://lwn.net/Articles/145980/>) from tests he had run. His initial test, involving an idle system, showed that power consumption varied from 7.59 watts with a 100Hz timer frequency to 8.15W at 1000Hz. [A subsequent test](<https://lwn.net/Articles/145982/>) with KDE running showed a smaller savings, especially when `artsd` was running. 

These results have given ammunition to both sides. Advocates of a low HZ value see the potential for a half-watt savings as worthwhile. Those who want HZ to be high see, instead, a change which makes the system less effective for them while yielding minimal advantages in real-world use. 

If there is a consensus on this issue, it would appear to be that the real solution is the [dynamic tick patch](<http://lwn.net/Articles/138969/>). By causing timer interrupts to happen only when there is actually something to be done, the kernel can simultaneously support higher-resolution timers and reduce the actual incidence of timer interrupts. No commitments have been made, but there seems to be a widely-held opinion that the dynamic tick patch will be merged once it has been sufficiently cleaned up; some architectures (e.g. ARM) already have it. To that end, Con Kolivas has posted [a reworked version](<http://lwn.net/Articles/145880/>) of that patch for review. 

If this patch is to be merged soon, it has been asked, why make a change to HZ in the mean time? No answers to that question have been posted. It is true that the lower value of HZ has been in the mainline for some time (and in -mm for even longer) and the number of real complaints has been small. In the absence of problems noted by a wider group of testers, the default value of 250 for HZ seems likely to persist into the final 2.6.13 release. It remains to be seen, however, what value the distributors will pick for the kernels they ship.
