---
title: The source of the e1000e corruption bug
url: https://lwn.net/Articles/304105/
date: "October 21, 2008"
category: "Ftrace; Releases-2.6.27"
author: "By Jonathan Corbet October 21, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 21, 2008

When LWN [last looked](<http://lwn.net/Articles/301251/>) at the e1000e hardware corruption bug, the source of the problem was, at best, unclear. Problems within the driver itself seemed like a likely culprit, but it did not take long for those chasing this problem to realize that they needed to look further afield. For a while, the X server came under scrutiny, as did a number of other system components. When the real problem was found, though, it turned out to be a surprise for everybody involved. 

Tracking down intermittent problems is hard. When those problems result in the destruction of hardware, finding them is even harder. Even the most dedicated testers tend to balk when faced with the prospect of shipping their systems back to the manufacturer for repairs. So the task of finding this issue fell to Intel; engineers there locked themselves into a lab with a box full of e1000e adapters and set about bisecting the kernel history to identify the patch which caused the problem. Some time (and numerous fried adapters) later, the bisection process turned up an unlikely suspect: the ftrace tracing framework. 

Developers working on tracing generally put a lot of effort into minimizing the impact of their code on system performance. Every last bit of runtime overhead is scrutinized and eliminated if at all possible. As a general rule, bricking the hardware is a level of overhead which goes well beyond the acceptable parameters. So the ftrace developers, once informed of the bisection result, put in some significant work of their own to figure out what was going on. 

One of the features offered by ftrace is a simple function-call tracing operation; ftrace will output a line with the called function (and its caller) every time a function call is made. This tracing is accomplished by using the venerable profiling mechanism built into gcc (and most other Unix-based compilers). When code is compiled with the `-pg` option, the compiler will place a call to `mcount()` at the beginning of every function. The version of `mcount()` provided by ftrace then logs the relevant information on every call. 

As noted above, though, tracing developers are concerned about overhead. On most systems, it is almost certain that, at any given time, nobody will be doing function call tracing. Having all those `mcount()` calls happening anyway would be a measurable drag on the system. So the ftrace hackers looked for a way to eliminate that overhead when it is not needed. A naive solution to this problem might look something like the following. Rather than put in an unconditional call to `mcount()`, get gcc to add code like this: 

```
if (function_tracing_active)
            mcount();
```

But the kernel makes a _lot_ of function calls, so even this version will have a noticeable overhead; it will also bloat the size of the kernel with all those tests. So the favored approach tends to be different: run-time patching. When function tracing is not being used, the kernel overwrites all of the `mcount()` calls with no-op instructions. As it happens, doing nothing is a highly optimized operation in contemporary processors, so the overhead of a few no-ops is nearly zero. Should somebody decide to turn function tracing on, the kernel can go through and patch all of those `mcount()` calls back in. 

Run-time patching can solve the performance problem, but it introduces a new problem of its own. Changing the code underneath a running kernel is a dangerous thing to do; extreme caution is required. Care must be taken to ensure that the kernel is not running in the affected code at the time, processor caches must be invalidated, and so on. To be safe, it is necessary to get all other processors on the system to stop and wait while the patching is taking place. The end result is that patching the code is an expensive thing to do. 

The way ftrace was coded was to patch out every `mcount()` call point as it was discovered through an actual call to `mcount()`. But, as noted above, run-time patching is very expensive, especially if it is done a single function at a time. So ftrace would make a list of `mcount()` call sites, then fix up a bunch of them later on. In that way, the cost of patching out the calls was significantly reduced. 

The problem now is that things might have changed between the time when an `mcount()` call is noticed and when the kernel gets around to patching out the call. It would be very unfortunate if the kernel were to patch out an `mcount()` call which no longer existed in the expected place. To be absolutely sure that unrelated data was not being corrupted, the ftrace code used the `cmpxchg` operation to patch in the no-ops. `cmpxchg` atomically tests the contents of the target memory against the caller's idea of what is supposed to be there; if the two do not match, the target location will be left with its old value at the end of the operation. So the no-ops will only be written to memory if the current contents of that memory are a call to `mcount()`. 

This all seems pretty safe, except that it fell down in one obscure, but important case. One obvious place where an `mcount()` call could go away is in loadable modules. This can happen if the module is unloaded, of course, but there is another important case too: any code marked as initialization code will be removed once initialization is complete. So a module's initialization function (and any other code marked `__init`) could leave a dangling reference in the "`mcount()` calls to be patched out" list maintained by ftrace. 

The final piece of this puzzle comes from this little fact: on 32-bit architectures, memory returned from `vmalloc()` and `ioremap()` share the same address space. Both functions create mappings to memory from the same range of addresses. Space for loadable modules is allocated with `vmalloc()`, so all module code is found within this shared address space. Meanwhile, the e1000e driver uses `ioremap()` to map the adapter's I/O memory and NVRAM into the kernel's address space. The end result is this fatal sequence of events: 

  1. A module is loaded into the system. As part of the module's initialization, a number of `mcount()` calls are made; these call sites are noted for later patching. 

  2. Module initialization completes, and the module's `__init` functions are removed from memory. The address space they occupied is freed up for future use. 

  3. The e1000e driver maps its I/O memory and NVRAM into the address range recently occupied by the above-mentioned initialization code. 

  4. Ftrace gets around to patching out the accumulated list of `mcount()` calls. But some of those "calls" are now, actually, I/O memory belonging to the e1000e device. 

Remember that the ftrace code was very careful in its patching, using `cmpxchg` to avoid overwriting anything which is not an `mcount()` call. But, as Steven Rostedt noted in [his summary of the problem](<http://lwn.net/Articles/303390/>): 

The cmpxchg could have saved us in most cases (via luck) - but with ioremap-ed memory that was exactly the wrong thing to do - the results of cmpxchg on device memory are undefined. (and will likely result in a write) 

The end result is a write to the wrong bit of I/O memory - and a destroyed device. 

In hindsight, this bug is reasonably clear and understandable, but it's not at all surprising that it took a long time to find. One should note that there were, in fact, two different bugs here. One of them is ftrace's attempt to write to a stale pointer. But the other one was just as important: the e1000e driver should never have left its hardware configured in a mode where a single stray write could turn it into a brick. One never knows where things might go wrong; hardware should never be left in such a vulnerable state if it can be helped. 

The good news is that both bugs have been fixed. The e1000e hardware was locked down before 2.6.27 was released, and the 2.6.27.1 update disables the dynamic ftrace feature. The ftrace code has been significantly rewritten for 2.6.28; it no longer records `mcount()` call sites on the fly, no longer uses `cmpxchg`, and, one hopes, is generally incapable of creating such mayhem again.
