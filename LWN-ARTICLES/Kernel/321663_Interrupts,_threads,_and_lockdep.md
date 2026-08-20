---
title: "Interrupts, threads, and lockdep"
url: https://lwn.net/Articles/321663/
date: "March 3, 2009"
category: Interrupts; Lockdep
author: "By Jonathan Corbet March 3, 2009"
---

By **Jonathan Corbet**  
March 3, 2009

Felipe Balbi recently posted [a driver called twl4030-pwrbutton](<http://lwn.net/Articles/321385/>), which generates input events when somebody hits a power button connected through a twl4030 i2c controller. It is, in many ways, a standard driver; Felipe certainly did not expect to see a long and acrimonious discussion result from its posting. But that's what ensued. Over the course of this discussion, the participants were able to outline some problems with how interrupts are handled on Linux systems, along with a potential solution. 

Things started when Andrew Morton questioned the following bit of code, found in the driver's interrupt handler: 

```
#ifdef CONFIG_LOCKDEP
    	/* WORKAROUND for lockdep forcing IRQF_DISABLED on us, which
    	 * we don't want and can't tolerate.  Although it might be
    	 * friendlier not to borrow this thread context...
    	 */
    	local_irq_enable();
        #endif
```

Workarounds of this variety do tend to catch the attention of diligent reviewers. Understanding this one requires just a bit of background. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Back in the Good Old Days, the Linux kernel had "fast" and "slow" interrupt handlers; the main difference between the two is that "fast" handlers ran with further interrupts disabled, while "slow" handlers were run with interrupts enabled. Over time, the distinction between the two types has faded; faster, smarter hardware and greater use of software interrupts and tasklets have made the execution time of most well-written interrupt handlers essentially irrelevant. So most driver authors do not even think much about whether they are writing a "fast" or a "slow" handler, even though the distinction still exists. Unless a driver passes the `IRQF_DISABLED` flag when requesting its interrupt line, its interrupt handler will be called with interrupts enabled. 

"Lockdep" is the [kernel lock validator](<http://lwn.net/Articles/185666/>), which, when enabled, creates a detailed model of how locks are used in the kernel. This model can be used to find potential deadlocks and other problems. [According to Ingo Molnar](<https://lwn.net/Articles/321670/>), lockdep has been quite effective: 

You might also have noticed that over the past 2-3 years the term "hard lockup" in regression reports has gone down by about an order of magnitude - and much of that can be attributed to the lockdep coverage we have in place. 

It turns out, though, that the lockdep developers made one significant, simplifying assumption: all interrupt handlers were to be invoked with interrupts disabled. When lockdep is enabled, in fact, the generic interrupt handling layer forces this condition, regardless of whether any specific handler was registered with the `IRQF_DISABLED` flag. Lockdep has worked this way for some time, and complaints have been scarce. But, as can be seen from the patch cited above, "scarce" is not the same as "nonexistent." 

Drivers for i2c-connected devices operate under a number of interesting constraints, mostly forced by the fact that the i2c "bus" is, in reality, a slow, two-wire serial interface. So even "fast" operations like reading a device register are, in fact, slow on i2c devices; they are slow enough that the process involved should sleep while waiting for the result. That is a bit of a problem for i2c interrupt handlers, since they need to access device registers, but they cannot sleep. 

The result is that a number of i2c drivers have implemented what is, in effect, a threaded interrupt handler mechanism. The "real" interrupt handler simply masks the interrupt and wakes up the thread, which then does the real work of talking to the device. In the case of the twl4030 driver, this threaded implementation has been done in a relatively formal manner in which the device interrupt handlers are invoked - from within a special-purpose kernel thread - by way of the generic IRQ layer itself. These threaded handlers do not expect to run with interrupts disabled - indeed, they cannot run that way - but the generic IRQ code will, when lockdep is enabled, turn off interrupts anyway. That is why this patch takes pains to turn them back on when lockdep is being used. 

Peter Zijlstra's response to this discussion was to post [a patch forcing `IRQF_DISABLED` for all drivers](<http://lwn.net/Articles/321386/>). His position is that no interrupt handlers should be run with interrupts enabled. Doing so invites kernel stack overruns if too many nested interrupts come in; it also, he says, encourages the notion that it's OK for interrupt handlers to be slow. Additionally, he says, drivers must already be able to run their handlers with interrupts disabled, since another driver may disable interrupts on a shared interrupt line. So, he says, it makes no sense to "fix" lockdep for handlers which want interrupts to be enabled; instead, the always-disabled assumption built into lockdep should be made part of the system as a whole. 

The response to this patch was somewhat sympathetic, at least in a general sense. Making `IRQF_DISABLED` be the default situation makes sense for most devices. But there really are drivers which need their interrupt handlers to run with interrupts enabled; IDE drivers using programmed I/O are one example. If those interrupt handlers are given exclusive control over the system, other devices will see unacceptable latencies and start to fail operations or drop data. So any change of this nature must be done carefully, and it must remain possible to run some handlers with interrupts enabled. 

And, of course, forcing `IRQF_DISABLED` does nothing to fix the twl4030 problem. 

The real solution is to have general support for threaded interrupt handlers. The realtime preemption tree has supported threaded handlers for quite some time; more recently, [a variant of the threaded handlers patch](<http://lwn.net/Articles/302043/>) was posted for mainline consideration. There are a lot of advantages to threaded handlers beyond their applicability to the problems discussed here; threaded handlers can improve latencies, allow interrupt handlers to be prioritized, and, someday, perhaps allow the removal of software interrupts altogether. So it seems like there would be value in getting this code merged. 

To that end, Thomas Gleixner has come back with [a new version of the threaded handlers patch](<http://lwn.net/Articles/320865/>). The API looks much like it did in the previous posting, though it could change in response to [some review comments](<https://lwn.net/Articles/321673/>) made this time around. In essence, this infrastructure allows a driver to register a "quick handler" to acknowledge (and mask) an interrupt; there would also be a regular handler which could be called in either hard interrupt or process context, depending on the quick handler's return value. The API allows drivers to continue to work unmodified, or they can be converted over to threaded handlers. 

David Brownell, the leading critic of lockdep's behavior and the idea of disabling interrupts for all handlers, seems to agree that the threaded interrupt handler infrastructure should be able to solve the i2c problem. All threaded handlers will, by necessity, run with interrupts enabled, so the primary difficulty goes away. David would like to see some changes made to better support the chaining of handlers that is typically needed in such situations, but it's not clear how many changes are really needed. 

In summary, threaded interrupt handlers seem likely to be the next technology to be merged from the realtime preemption tree. Just when that might happen remains to be seen, though. The request for some API changes may well slow things down a bit; there were also requests for example implementations of threaded handlers with more types of drivers. Satisfying those requests quickly enough to allow the code to be reviewed before the 2.6.30 merge window opens could be a bit of a challenge. So this code might just have to wait for one more development cycle; it would be surprising if it were to take longer than that, though.
