---
title: Avoiding retpolines with static calls
url: https://lwn.net/Articles/815908/
date: "March 26, 2020"
category: Retpoline; Static calls
author: "By Jonathan Corbet March 26, 2020"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 26, 2020

January 2018 was a sad time in the kernel community. The Meltdown and Spectre vulnerabilities had finally been disclosed, and the required workarounds hurt kernel performance in a number of ways. One of those workarounds — [retpolines](<https://support.google.com/faqs/answer/7625886>) — continues to cause pain, with developers going out of their way to avoid indirect calls, since they must now be implemented with retpolines. In some cases, though, there may be a way to avoid retpolines and regain much of the lost performance; after a long gestation period, the "static calls" mechanism may finally be nearing the point where it can be merged upstream. 

Indirect calls happen when the address of a function to be called is not known at compile time; instead, that address is stored in a pointer variable and used at run time. These indirect calls, as it turns out, are readily exploited by speculative-execution attacks. Retpolines defeat these attacks by turning an indirect call into a rather more complex (and expensive) code sequence that cannot be executed speculatively. 

Retpolines solved the problem, but they also slow down the kernel, so developers have been keenly interested in finding ways to avoid them. A number of approaches have been tried; a few of which were [covered here](<https://lwn.net/Articles/774743/>) in late 2018. While some of those techniques have been merged, static calls have remained outside of the mainline. They have recently returned in the form of [this patch set](<https://lwn.net/ml/linux-kernel/20200324135603.483964896@infradead.org/>) posted by Peter Zijlstra; it contains the work of others as well, in particular Josh Poimboeuf, who posted the original static-call implementation. 

An indirect call works from a location in writable memory where the destination of the jump can be found. Changing the destination of the call is a matter of storing a new address in that location. Static calls, instead, use a location in _executable_ memory containing a jump instruction that points to the target function. Actually executing a static call requires "calling" to this special location, which will immediately jump to the real target. The static-call location is, in other words, a classic code trampoline. Since both jumps are direct — the target address is found directly in the executable code itself — no retpolines are needed and execution is fast. 

Static calls must be declared before they can be used; there are two macros that can do that: 

```
#include <linux/static_call.h>
    
        DEFINE_STATIC_CALL(name, target);
        DECLARE_STATIC_CALL(name, target);
```

`DEFINE_STATIC_CALL()` creates a new static call with the given `name` that initially points at the function `target()`. `DECLARE_STATIC_CALL()`, instead, declares the existence of a static call that is defined elsewhere; in that case, `target()` is only used for type checking the calls. 

Actually calling a static call is done with: 

```
static_call(name)(args...);
```

Where `name` is the name used to define the call. This will cause a jump through the trampoline to the target function; if that function returns a value, `static_call()` will also return that value. 

The target of a static call can be changed with: 

```
static_call_update(name, target2);
```

Where `target2()` is the new target for the static call. Changing the target of a static call requires patching the code of the running kernel, which is an expensive operation. That implies that static calls are only appropriate for settings where the target will change rarely. 

One such setting can be found in the patch set: tracepoints. Activating a tracepoint itself requires code patching. Once that is done, the kernel responds to a hit on a tracepoint by iterating through a linked list of callback functions that have been attached there. In almost every case, though, there will only be one such function. [This patch](<https://lwn.net/ml/linux-kernel/20200324142246.005164711@infradead.org/>) in the series optimizes that case by using a static call for the single-function case. Since the intent behind tracepoints is to minimize their overhead to the greatest extent possible, use of static calls makes sense there. 

This patch set also contains a further optimization not found in the original. Jumping through the trampoline is much faster than using a retpoline, but it is still one more jump than is strictly necessary. So [this patch](<https://lwn.net/ml/linux-kernel/20200324142245.694414364@infradead.org/>) causes static calls to store the target address directly into the call site(s), eliminating the need for the trampoline entirely. Doing so may require changing multiple call sites, but most static calls are unlikely to have many of those. It also requires support in the `objtool` tool to locate those call sites during the kernel build process. 

The end result of this work [appears to be](<https://lwn.net/ml/linux-kernel/20181126155405.72b4f718@gandalf.local.home/>) a significant reduction in the cost of the Spectre mitigations when using tracepoints — a slowdown of just over 4% drops to about 1.6%. It has been through a number of revisions, as well as some improvements to the underlying text-patching code, and appears to be about ready. Chances are that static calls will go upstream in the near future.
