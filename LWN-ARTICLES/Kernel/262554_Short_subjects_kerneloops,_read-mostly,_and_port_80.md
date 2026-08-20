---
title: "Short subjects: kerneloops, read-mostly, and port 80"
url: https://lwn.net/Articles/262554/
date: "December 18, 2007"
category:   read mostly; Debugging; Device drivers
author: "By Jonathan Corbet December 18, 2007"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 18, 2007

**Kerneloops**. Triage is an important part of a kernel developer's job. A project as large and as widely-used as the kernel will always generate more bug reports than can be realistically addressed in the amount of time which is available. So developers must figure out which reports are most deserving of their attention. Sometimes the existence of an irate, paying customer makes this decision easy. Other times, though, it is a matter of making a guess at which bugs are affecting the largest numbers of users. And that often comes down to how many different reports have come in for a given problem. 

Of course, counting reports is not the easiest thing to do, especially if they are not all sent to the same place. In an attempt to make this process easier, Arjan van de Ven has [announced](<https://lwn.net/Articles/262555/>) a new site at [kerneloops.org](<http://kerneloops.org>). Arjan has put together some software which scans certain sites and mailing lists for posted kernel oops output; whenever a crash is found, it is stuffed into a database. Then an attempt is made to associate reports with each other based on kernel version and the call trace; from that, a list of the most popular ways to crash can be created. As of this writing, the current fashion for kernel oopses would appear to be in `ieee80211_tx()` in current development kernels. Some other information is stored with the trace; in particular, it is possible to see what the oldest kernel version associated with the problem is. 

This is clearly a useful resource, but there are a couple of problems which make it harder to do the job properly. One is that there is no distinctive marker which indicates the end of an oops listing, so the scripts have a hard time knowing where to stop grabbing information. The other is that multiple reports of the same oops can artificially raise the count for a particular crash. The solution to both problems is to place a marker at the end of the oops output which includes a random UUID generated at system boot time. Patches to this effect are circulating, though getting the random number into the output turns out to be a little harder than one might have expected. So, for 2.6.24, the "random" number may be all zeroes, with the real problem to be solved in 2.6.25. 

**Read-mostly**. Anybody who digs through kernel source for any period of time will notice a number of variables declared in a form like this: 

```
static int __read_mostly ignore_loglevel;
```

The `__read_mostly` attribute says that accesses to this variable are usually (but not always) read operations. There were some questions recently about why this annotation is done; the answer is that it's an important optimization, though it may not always be having the effect that developers are hoping for. 

As is well described in [What every programmer should know about memory](<http://lwn.net/Articles/250967/>), proper use of processor memory caches is crucial for optimal performance. The idea behind `__read_mostly` is to group together variables which are rarely changed so they can all share cache lines which need not be bounced between processors on multiprocessor systems. As long as nobody changes a `__read_mostly` variable, it can reside in a shared cache line with other such variables and be present in cache (if needed) on all processors in the system. 

The read-mostly attribute generally works well and yields a measurable performance improvement. There are concerns, though, that this feature could be over-used. Andrew Morton [expressed it](<https://lwn.net/Articles/262557/>) this way: 

So... once we've moved all read-mostly variables into __read_mostly, what is left behind in bss? All the write-often variables. All optimally packed together to nicely maximise cacheline sharing. 

Combining frequently-written variables into shared cache lines is a good way to maximize the bouncing of those cache lines between processors - which would be bad for performance. So over-aggressive segregation of read-mostly variables to minimize cache line bouncing could have the opposite of the desired effect: it could make the kernel's cache behavior worse. 

The better way, says Andrew, would have been to create a "read often" attribute for variables which are _frequently_ used in a read-only mode. That would leave behind the numerous read-rarely variables to serve as padding keeping the write-often variables nicely separated from each other. Thus far, patches to make this change have not been forthcoming. 

**I/O port delays**. The functions provided by the kernel for access to I/O ports have long included versions which insert delays. A driver would normally read a byte from a port with `inb()`, but `inb_p()` could be used if an (unspecified) short delay was needed after the operation. A look through the driver tree shows that quite a few drivers use the delayed versions of the I/O port accessors, even though, in many cases, there is no real need for that delay. 

This delay is implemented (on x86 architectures) with a write to I/O port 80\. There is generally no hardware listening for an I/O operation on that port, so this write has the sole effect of delaying the processor while the bus goes through an abortive attempt to execute the operation. It is an operation with reasonably well-defined semantics, and it has worked for Linux for many years. 

Except that now, [it seems](<https://lwn.net/Articles/262559/>), this technique no longer works on a small subset of x86_64 systems. Instead, the write to port 80 will, on occasion, freeze the system hard; this, in turn, generates a rather longer delay than was intended. One could imagine the creation of an elaborate mechanism for restarting I/O operations after the user resets the system, but the kernel developers, instead, chose to look for alternative ways of implementing I/O delays. 

In almost every case, the alternative form of the delay is a call to `udelay()`. The biggest problem here is that `udelay()` works by sitting in a tight loop; it cannot know how many times to go through the loop until the speed of the processor has been calibrated. That calibration happens reasonably early in the boot process, but there are still tasks to be performed - including I/O port operations - first. This problem is being worked around by removing some delayed operations from the early setup code, but some developers [worry](<https://lwn.net/Articles/262562/>) that it will never be possible to get them all. It has been suggested that the kernel could just assume it's running on the fastest-available processor until the calibration happens, but, beyond being somewhat inelegant, that could significantly slow the bootstrap process on slower machines - all of which work just fine with the current code. 

The real solution is to simply get rid of almost all of the delayed I/O port operations. Very few of them are likely to be needed with any hardware which still works. In some cases, what may really be going on is that the delays are being used to paper over driver bugs - such as failing to force a needed PCI write out by doing a read operation. Just removing the delays outright would probably cause instability in unpredictable places - not a result most developers are striving for. So the task of cleaning up those calls will have to be done carefully over time. Meanwhile, the use of port 80 will probably remain unchanged for 2.6.24.
