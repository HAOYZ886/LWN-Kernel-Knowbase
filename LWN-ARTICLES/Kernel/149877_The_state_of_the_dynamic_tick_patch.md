---
title: The state of the dynamic tick patch
url: https://lwn.net/Articles/149877/
date: "August 31, 2005"
category: Timer frequency
author: 
---

The configurable timer interrupt frequency patch, part of the 2.6.13 kernel, led to a certain amount of controversy over the optimal default value. That default is 250 Hz, but there arguments in favor of both increasing and decreasing that value. There was no consensus on what the default should really be, but there _is_ a certain amount of agreement that the real solution is to merge the [dynamic tick patch](<https://lwn.net/Articles/138969/>). By varying the timer interrupt frequency in response to the actual system workload, the dynamic tick approach should be able to satisfy most users. 

Now that patches are being merged for 2.6.14, the obvious question came up: will dynamic tick be one of them? The answer, it seems, is almost certainly "no." This patch, despite being around in one form or another for years, is still not quite ready. 

One issue, apparently, is that systems running with dynamic tick tend to boot slowly, and nobody has yet figured out why. The problem can be masked by simply waiting until the system has booted before turning on dynamic tick, but that solution appeals to nobody. Until this behavior is understood, there will almost certainly be opposition to the merging of this patch. 

Another problem with the current patch is that it does not work particularly well on SMP systems. It requires that _all_ CPUs go idle before the timer interrupt frequency can be reduced. But an SMP system may well have individual CPUs with no work to do while others are busy; such a situation could come up fairly often. Srivatsa Vaddagiri is working on [a patch for SMP systems](<https://lwn.net/Articles/147370/>), but it is still a work in progress and has not received widespread testing. 

The end result is that dynamic tick is unlikely to come together in time to get into 2.6.14; the window for merging of patches of this magnitude is supposed to close within a week or so. So this patch will be for 2.6.15 at the earliest. If the revised development process works as planned, 2.6.15 should not be all that far away. Hopefully.
