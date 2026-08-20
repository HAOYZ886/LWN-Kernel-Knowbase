---
title: Irked by NO_IRQ
url: https://lwn.net/Articles/470820/
date: "December 7, 2011"
category: "Architectures-Arm; Interrupts"
author: "By Jonathan Corbet December 7, 2011"
---

By **Jonathan Corbet**  
December 7, 2011

The kernel has always used small integer numbers to represent interrupt (IRQ) lines internally; those numbers usually correspond to the numbers of the interrupt lines feeding into the CPU, though it need not be that way. The kernel has also traditionally used the value zero to indicate that there is no IRQ assignment at all - except in the places where it hasn't. That inconsistency has, after many years, popped up to bite the ARM architecture, with the result that there may have to be a big scramble to fix things for the 3.3 merge window. 

While the x86 architecture and core interrupt code have used zero for "no IRQ," various architectures have made other decisions, often using a value of `-1` instead. The reason for making such a choice is straightforward: many architectures have a valid IRQ line numbered zero. In that situation, a few options present themselves. That inconveniently-numbered IRQ can be hidden away where nobody actually sees it; that is what x86 does on systems where the timer interrupt shows up as zero. Another option is to remap that interrupt line to a different number in software so that IRQ 0 never appears outside of the core architecture code. Or the architecture can treat zero as a valid IRQ number and define a symbol like `NO_IRQ` to a different value; a number of architectures have done that over time, though some have switched to zero in recent years. But the ARM architecture still uses -1 as its `NO_IRQ` value. 

It is worth noting that `NO_IRQ` has been a topic of discussion at various times over the years. The earliest appears to have been in response to [this patch from Matthew Wilcox](<http://linux.derkeiler.com/Mailing-Lists/Kernel/2005-11/6937.html>), posted in 2005. At that time, Linus [made it clear](<http://linux.derkeiler.com/Mailing-Lists/Kernel/2005-11/7628.html>) that he saw zero as the only valid `NO_IRQ` value. His reasoning was that code reading: 

```
if (!dev->irq)
    	/* handle the no-IRQ case */
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

is easier to read and more efficient to execute than comparisons against some other value. Beyond that, IRQ numbers were (and are) represented by unsigned variables in many parts of the kernel; values like `-1` are clearly an awkward fit in such situations. Given that, he said: 

So my suggested (very _strongly_ suggested) solution is for people to just consider "irq" to be a cookie, with the magical value 0 meaning "not there" but no inherent meaning otherwise. That just solves all the fundamentally hard problems, and has no fundamental problems of its own. 

In the intervening six years, most architectures have followed that suggestion, but ARM has not. That has mattered little thus far, but, as ARM's popularity grows and the amount of code shared with the rest of the kernel grows with it, that inconsistency is leading to problems. Some recent changes to the device tree code threaten to break ARM entirely (for some subplatforms, anyway) if things are not changed. 

How hard will things be to fix? Alan Cox offered [this suggestion](<https://lwn.net/Articles/470823/>): 

I have so little sympathy over this that you'll need a quantum physicist to measure it.... You've had years to fix it. If I were you I'd delete NO_IRQ from your tree, type make and get it done. It's not even a big job to clean it out. 

As it happens, though, it's not quite that easy. The core ARM code will need to remap IRQ 0 in places where it is a valid number. Arguably the most unpleasant problem, though, was [pointed out](<https://lwn.net/Articles/470824/>) by Dave Martin: 

Unfortunately, NO_IRQ is often not spelled "NO_IRQ". It looks like the assumption "irq < 0 === no irq" may be quite a lot more widespread than "NO_IRQ === no irq". Since there's no specific thing we can grep for (and simply due to volume) finding all such instances may be quite a bit harder. 

So it will be necessary to find all of the places where the code assumes that a negative IRQ number means "no interrupt," places where `-1` has been hard coded, and so on. One can argue (as [ARM maintainer Russell King has](<https://lwn.net/Articles/470826/>)) that, as long as `request_irq(0)` fails and physical IRQ 0 is handled properly, most of that code will do the right thing without being changed. But there is still a fair amount of stuff to clean up. 

Putting it into perspective, though: the work required to fix `NO_IRQ` in the ARM tree is likely to be tiny relative to the general cleanup that is happening there. A bit of churn, some drivers to fix, and ARM should come into agreement with the rest of the kernel on the way to indicate a missing interrupt. That, in turn, will increase robustness and allow the sharing of more code with the rest of the kernel - not something to get too terribly irked about.
