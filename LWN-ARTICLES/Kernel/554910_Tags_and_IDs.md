---
title: Tags and IDs
url: https://lwn.net/Articles/554910/
date: "June 19, 2013"
category: Block layer; idr
author: "By Jonathan Corbet June 19, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 19, 2013

Our recent coverage of the [multiqueue block layer work](<https://lwn.net/Articles/552904/>) touched on a number of the changes needed to enable the kernel to support devices capable of handling millions of I/O operations per second. But, needless to say, there are plenty of additional details that must be handled. One of them, the allocation of integer tags to identify I/O requests, seems like a relatively small issue, but it has led to an extensive discussion that, in many ways, typifies how kernel developers look at proposed additions. 

Solid-state storage devices will only achieve their claimed I/O rates if the kernel issues many I/O operations in parallel. That allows the device to execute the requests in an optimal order and to exploit the parallelism inherent in having multiple banks of flash storage. If the kernel is not to get confused, though, there must be a way for the device to report the status of specific operations to the kernel; that is done by assigning a tag (a small integer value) to each request. Once that is done, the device can report that, say, request #42 completed, and the kernel will know which operation is done. 

If the device is handling vast numbers of operations per second, the kernel will somehow have to come up with an equal number of tags. That suggests that tag allocation must be a fast operation; even a small amount of overhead starts to really hurt when it is repeated millions of times every second. To that end, Kent Overstreet has proposed the merging of a [per-CPU tag allocator](<https://lwn.net/Articles/554097/>), a new module with a simple task: allocate unique integers within a given range as quickly as possible. 

The interface is relatively straightforward. A "tag pool," from which tags will be allocated, can be declared this way: 

```
#include <linux/percpu-tags.h>
    
        struct percpu_tag_pool pool;
```

Initialization is then done with: 

```
int percpu_tag_pool_init(struct percpu_tag_pool *pool, unsigned long nr_tags);
```

where `nr_tags` is the number of tags to be contained within the pool. Upon successful initialization, zero will be returned to the caller. 

The actual allocation and freeing of tags is managed with: 

```
unsigned percpu_tag_alloc(struct percpu_tag_pool *pool, gfp_t gfp);
        void percpu_tag_free(struct percpu_tag_pool *pool, unsigned tag);
```

A call to `percpu_tag_alloc()` will allocate a tag from the given `pool`. The only use for the `gfp` argument is to be checked for the `__GFP_WAIT` flag; if (and only if) that flag is present, the function will wait for an available tag if need be. The return value is the allocated tag, or `TAG_FAIL` if no allocation is possible. 

The implementation works by maintaining a set of per-CPU lists of available tags; whenever possible, `percpu_tag_alloc()` will simply take the first available entry from the local list, avoiding contention with other CPUs. Failing that, it will fall back to a global list of tags, moving a batch of tags to the appropriate per-CPU list. Should the global list be empty, `percpu_tag_alloc()` will attempt to steal some tags from another CPU or, in the worst case, either wait for an available tag or return `TAG_FAIL`. Most of the time, with luck, tag allocation and freeing operations can be handled entirely locally, with no contention or cache line bouncing issues. 

The attentive reader might well be thinking that the API proposed here looks an awful lot like the [IDR subsystem](<https://lwn.net/Articles/536293/>), which also exists to allocate unique integer identifiers. That is where the bulk of the complaints came from; Andrew Morton, in particular, was [unhappy](<https://lwn.net/Articles/554918/>) that no apparent attempt had been made to adapt IDR before launching into a new implementation: 

The worst outcome here is that idr.c remains unimproved and we merge a new allocator which does basically the same thing. 

The best outcome is that idr.c gets improved and we don't have to merge duplicative code. 

So please, let's put aside the shiny new thing for now and work out how we can use the existing tag allocator for these applications. If we make a genuine effort to do this and decide that it's fundamentally hopeless then this is the time to start looking at new implementations. 

The responses from Kent (and from Tejun Heo as well) conveyed their belief that IDR is, indeed, fundamentally hopeless for this use case. The IDR code is designed for the allocation of identifiers, so it works a little differently: the lowest available number is always returned and the number range is expanded as needed. The lowest-number guarantee, in particular, forces a certain amount of cross-CPU data sharing, putting a limit on how scalable the IDR code can be. The IDR API also supports storing (and quickly looking up) a pointer value associated with each ID, a functionality not needed by users of tags. As Tejun [put it](<https://lwn.net/Articles/554923/>), even if the two allocators were somehow combined, there would still need to be two distinct ways of using it, one with allocation ordering guarantees, and one for scalability. 

Andrew proved hard to convince, though; he suggested that, perhaps, tag allocation could be implemented as some sort of caching layer on top of IDR. His position appeared to soften a bit, though, when Tejun [pointed out](<https://lwn.net/Articles/554925/>) that the I/O stack already has several tag-allocation implementations, ""and most, if not all, suck"". The per-CPU tag allocator could replace those implementations with common code, reducing the amount of duplication rather than increasing it. Improvements of that sort can work wonders when it comes to getting patches accepted. 

Things then took another twist when Kent posted [a rewrite of the IDA module](<https://lwn.net/Articles/555189/>) as the basis for a new attempt. "IDA" is a variant of IDR that lacks the ability to store pointers associated with IDs; it uses many of the IDR data structures but does so in a way that is more space-efficient. Kent's rewrite turns IDA into a separate layer, with the eventual plan of rewriting IDR to sit on top. Before doing that, though, he implemented [a new per-CPU ID allocator](<https://lwn.net/Articles/555190/>) implementing the API described above on top of the new IDA code. The end result should be what Andrew was asking for: a single subsystem for the allocation of integer IDs that accommodates all of the known use cases. 

All this may seem like an excessive amount of discussion around the merging of a small bit of clearly-useful code that cannot possibly cause bugs elsewhere in the kernel. But if there is one thing that the community has learned over the years, it's that kernel developers are far less scalable than the kernel itself. Duplicated code leads to inferior APIs, more bugs, and more work for developers. So it's worth putting some effort into avoiding the merging of duplicated functionality; it is work that will pay off in the long term — and the kernel community is expecting to be around and maintaining the code for a long time.
