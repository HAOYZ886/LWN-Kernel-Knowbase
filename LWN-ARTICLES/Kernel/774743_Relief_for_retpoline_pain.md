---
title: Relief for retpoline pain
url: https://lwn.net/Articles/774743/
date: "December 14, 2018"
category: Retpoline
author: "By Jonathan Corbet December 14, 2018"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 14, 2018

Indirect function calls — calls to a function whose address is stored in a pointer variable — have never been blindingly fast, but the Spectre hardware vulnerabilities have made things far worse. The indirect branch predictor used to speed up indirect calls in the CPU can no longer be used, and performance has suffered accordingly. The ["retpoline" mechanism](<https://support.google.com/faqs/answer/7625886>) was a brilliant hack that proved faster than the hardware-based solutions that were tried at the beginning. While retpolines took a lot of the pain out of Spectre mitigation, experience over the last year has made it clear that they still hurt. It is thus not surprising that developers have been looking for alternatives to retpolines; several of them have shown up on the kernel lists recently. 

The way to make an indirect call faster is to replace it with a direct call; that renders branch prediction unnecessary. Of course, if a direct call would have sufficed in any given situation, the developer would have used it rather than an indirect call, so this replacement is not always straightforward. All of the proposed solutions to retpoline overhead strive to do that replacement in one way or another, though; they vary from the simple to the complex. 

#### Speeding up DMA operations

The simplest method is often the best; that is the approach taken in [Christoph Hellwig's patch set](<https://lwn.net/ml/linux-kernel/20181207190720.18517-1-hch@lst.de/>) speeding up the DMA-mapping code. Setting up DMA buffers can involve a lot of architecture-specific trickery; the DMA mapping layer does its best to hide that trickery behind a common API. As is often the case in the kernel, the code in the middle uses a structure full of function pointers to direct a generic DMA call to the code that can implement it in any specific setting. 

It turns out, though, that the most common case for DMA mapping is the simplest: the memory is simply directly mapped in both the CPU's and the device's address space with no particular trickery required. Hellwig's work takes advantage of that fact by testing for this case and calling the direct-mapping support code directly rather than going through a function pointer. So, for example, code that looks like this: 

```
addr = ops->map_page(...);
```

is transformed into something like: 

```
if (dma_is_direct(ops))
        	addr = dma_direct_map_page(...);
        else
        	addr = ops->map_page(...);
```

The cost of the `if` test is more than recouped in the direct-mapping case by avoiding the indirect function call (and it is tiny relative to the cost of that call in the other cases). Jesper Dangaard Brouer, who reported the performance hit in the DMA-mapping code, [expressed his happiness](<https://lwn.net/ml/linux-kernel/20181208170648.296fb8a1@redhat.com/>) at this change: ""my XDP performance is back"". Barring problems, this change seems likely to be merged sometime soon. 

#### Choosing from a list

In some situations, an indirect function call will end up invoking one out of a relatively small list of known functions; a variant of the above approach can be used to test for each of the known alternatives and call the correct function directly. [This patch set](<https://lwn.net/ml/linux-kernel/cover.1544032300.git.pabeni@redhat.com/>) from Paolo Abeni implements that approach with a simple set of macros. If a given variable `func` can point to either of `f1()` or `f2()`, the indirect call can be avoided with code that looks like this: 

```
INDIRECT_CALLABLE_DECLARE(f1(args...));
        INDIRECT_CALLABLE_DECLARE(f2(args...));
        /* ... */
        INDIRECT_CALL_2(func, f2, f1, args...);
```

This code will expand to something like: 

```
if (func == f1)
        	f1(args);
        else if (func == f2)
        	f2(args);
        else
        	(*func)(args);
```

Abeni's patch set is aimed at the network stack, so it contains some additional optimizations that can apply when the choice is between the IPv4 and IPv6 versions of a function. He claims a 10% or so improvement for a UDP generic receive offload (GRO) benchmark. Networking maintainer David Miller has [indicated a willingness](<https://lwn.net/ml/linux-kernel/20181206.222409.551374562843523036.davem@davemloft.net/>) to accept this work, though the current patch set [needs a couple of repairs](<https://lwn.net/ml/linux-kernel/20181206.222855.1688568512001921392.davem@redhat.com/>) before it can be merged. 

#### Static calls

Sometimes indirect calls reflect a mode of operation in the kernel that is not often changed; in such cases, the optimal approach might be to just turn the indirect call into a direct call and patch the code when the target must be changed. That is the approach taken by the [static calls patch set](<https://lwn.net/ml/linux-kernel/cover.1543200841.git.jpoimboe@redhat.com/>) from Josh Poimboeuf. 

Imagine a global variable `target` that can hold a pointer to either of `f1()` or `f2()`. This variable could be declared as a static call with a declaration like: 

```
DEFINE_STATIC_CALL(target, f1);
```

Initially, `target` will point to `f1()`. Changing it to point to `f2()` requires a call like: 

```
static_call_update(target, f2);
```

Actually calling the function pointed to by `target` is done with `static_call()`: 

```
static_call(target, args...);
```

Since changing the target of a call involves code patching, it is an expensive operation and should not be done often. One possible use case for static calls is tracepoints in the kernel, which can have an arbitrary function attached to them, but which are not often changed. Using a static call for that attachment can reduce the runtime overhead of enabling a tracepoint. 

This patch set has been through a couple of revisions so far. It implements two different mechanisms. The first tracks all call sites for each static call variable and patches each of them when the target changes; the second stores the target in a trampoline and all calls jump through there. The motivations for the two approaches are not spelled out, but one can imagine that the direct calls will be a little faster, while the trampoline will be quicker and easier to patch when the target changes. 

#### Relpolines/optpolines

A rather more involved and general-purpose approach can be seen in [this patch set](<https://lwn.net/ml/linux-kernel/20181018005420.82993-1-namit@vmware.com/>) posted by Nadav Amit in October. Rather than requiring developers to change indirect call sites by hand, Amit adds a new mechanism that optimizes indirect calls on the fly. 

The patch set uses some ""assembly macro magic"" to change how every retpoline injected into the kernel works; the new version contains both fast and slow paths. The fast path is a test and direct call to the most frequently called target (hopefully) from any retpoline, while the slow path is the old retpoline mechanism. In the normal production mode, the fast path should mitigate the retpoline overhead in a large fraction of the calls from that site. 

What makes this work interesting is the selection of the target for the fast path. Each "relpoline" (a name that was deemed too close to "retpoline" for comfort and which, as a result, may be renamed to something like "optpoline") starts out in a learning mode where it builds a hash table containing the actual target for each call that is made. After a sufficient number of calls, the most frequently called target is patched directly into the code, and the learning mode ends. To follow changing workloads, relpolines are put back into the learning mode after one minute of operation, a period that Amit says ""might be too aggressive"". 

This mechanism has the advantage of optimizing all indirect calls, not just the ones identified as a problem by a developer. It can also operate on indirect calls added in loadable modules at any point during the system's operation. The results, he says, are ""not bad""; they include a 10% improvement in an nginx benchmark. Even on a system with retpolines disabled, simply optimizing the indirect calls yields a 2% improvement for nginx. The downside, of course, is the addition of a fair amount of low-level complexity to implement this mechanism. 

Response to this patch set has been muted but generally positive. There are, though, lots of suggestions on the details of how this mechanism would work. There may be further optimizations to be had by storing more than one common target, for example. The learning mechanism can probably benefit from some improvement. There was also a suggestion to use a GCC plugin rather than the macro magic to insert the new call mechanism into the kernel. As a result, the patch set is still under development and will likely take some time yet to be ready. 

#### What's next

Various other developers have been working on the indirect call problem as well. Edward Cree, for example, has posted [a patch set](<https://lwn.net/ml/linux-kernel/cf6e9449-f3f6-e0fc-8096-eaba6b5a3b97@solarflare.com/>) adding a simple learning mechanism to static calls. Nearly one year after the Spectre vulnerability was disclosed, the development community is clearly still trying to do something about the performance costs the Spectre mitigations have imposed. 

The current round of fixes is trying to recover the performance lost when the indirect branch predictor was taken out of the picture. As Cree [put it](<https://lwn.net/ml/linux-kernel/7ab6063d-92ac-4708-d820-0cf175cf0f92@solarflare.com/>): ""Essentially we're doing indirect branch prediction in software because the hardware can't be trusted to get it right; this is sad"". Merging four different approaches (at least) to this problem may not be the best solution, especially since this particular vulnerability should eventually be fixed in the hardware, rendering all of the workarounds unnecessary. Your editor would not want to speculate on which of the above patches, if any, will make it into the mainline, though.
