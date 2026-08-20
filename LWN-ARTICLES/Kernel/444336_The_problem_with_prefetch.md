---
title: The problem with prefetch
url: https://lwn.net/Articles/444336/
date: "May 24, 2011"
category: Prefetch
author: "By Jonathan Corbet May 24, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 24, 2011

Over time, software developers tend to learn that micro-optimization efforts are generally not worthwhile, especially in the absence of hard data pointing out a specific problem. Performance problems are often not where we think they are, so undirected attempts to tweak things to make them go faster can be entirely ineffective. Or, indeed, they can make things worse. That is a lesson that the kernel developers have just relearned. 

At the kernel level, performance often comes down to cache behavior. Memory references which must actually be satisfied by memory are extremely slow; good performance requires that needed data be in a CPU cache much of the time. The kernel goes out of its way to use cache-hot memory when possible; there has also been some significant work put into tasks like reordering structures so that fields that are commonly accessed together are found in the same cache line. As a general rule, these optimizations have helped performance in measurable ways. 

Cache misses are often unavoidable, but it is sometimes possible to attempt to reduce their cost. If the kernel knows that it will be accessing memory at a particular location in the near future, it can use a CPU-specific prefetch instruction to begin the process of bringing the data into cache. This instruction is made available to kernel code via the generic `prefetch()` function; developers have made heavy use of it. Consider, for example, this commonly-used macro from `<linux/list.h>`: 

```
#define list_for_each(pos, head) \
    	for (pos = (head)->next; prefetch(pos->next), pos != (head); \
                pos = pos->next)
```

This macro (in a number of variants) is used to traverse a linked list. The idea behind the `prefetch()` call here is to begin the process of fetching the next entry in the list while the current entry is being processed. Hopefully by the time the next loop iteration starts, the data will have arrived - or, at least, it will be in transit. Linked lists are known to be cache-unfriendly data structures, so it makes sense that this type of optimization can help to speed things up. 

Except that it doesn't - at least, not on x86 processors. 

Andi Kleen may have been the first to question this optimization when he [tried to remove the prefetches from list operations](<https://lwn.net/Articles/404103/>) last September. His patch generated little discussion, though, and apparently fell through the cracks. Recently, Linus [did some profiling](<https://lwn.net/Articles/444344/>) on one of his favorite workloads (kernel builds) and found that the prefetch instructions were at the top of the ranking. Performing the prefetching cost time, and that time was not being repaid through better cache behavior; simply removing the `prefetch()` calls made the build go faster. 

Ingo Molnar, being Ingo, [jumped in](<https://lwn.net/Articles/444346/>) and did a week's worth of research in an hour or so. Using perf and a slightly tweaked kernel, he was able to verify that using the prefetch instructions caused a performance loss of about 0.5%. That is not a headline-inspiring performance regression, certainly, but this is an optimization which was supposed to make things go faster. Clearly something is not working the way that people thought it was. 

Linus pointed out one problem at the outset: his test involved a lot of traversals of singly-linked `hlist` hash table lists. Those lists tend to be short, so there is not much scope for prefetching; in fact, much of the time, the only prefetch attempted used the null pointer that indicates the end of the list. Prefetching with a null pointer seems silly, but it's also costly: evidently every such prefetch on x86 machines (and, seemingly, ARM as well) causes a translation lookaside buffer miss and a pipeline stall. Ingo measured this effect and came to the conclusion that each null prefetch cost about 20 processor cycles. 

Clearly, null prefetches are a bad idea. It would be nice if the CPU would simply ignore attempts to prefetch using a null pointer, but that's not how things are, so, as is often the case, one ends up trying to solve the problem in software instead. Ingo did some testing with a version of `prefetch()` which would only issue prefetch instructions for non-null pointers; that version did, indeed, perform better. But it still performed measurably worse than simply skipping the prefetching altogether. 

CPU designers are well aware of the cost of waiting for memory; they have put a great deal of effort into minimizing that cost whenever possible. Among other things, contemporary CPUs have their own memory prefetch units which attempt to predict which memory will be wanted next and start the process of retrieving it early. One thing Ingo noticed in his tests is that, even without any software prefetch operations, the number of prefetch operations run by the CPU was about the same. So the hardware prefetcher was busy during this time - and it was doing a better job than the software at deciding what to fetch. Throwing explicit prefetch operations into the mix, it seems, just had the effect of interfering with what the hardware was trying to do. 

Ingo summarized his results this way: 

So the conclusion is: prefetches are absolutely toxic, even if the NULL ones are excluded. 

One immediate outcome from this work is that, for 2.6.40 (or whatever it ends up being called), the `prefetch()` calls have been removed from linked list, hlist, and sk_buff list traversal operations - just like Andi Kleen tried to do in September. Chances are good that other prefetch operations will be removed as well. There will still be a place for `prefetch()` in the kernel, but only in specific situations where it can be clearly shown to help performance. As with other low-level optimizations (`likely()` comes to mind), tossing in a prefetch because it seems like it might help is often not the right thing to do. 

One other lesson to be found in this experience is that numbers matter. Andi was right when he wanted to remove these operations, but he did not succeed in getting his patch merged. One could come up with a number of reasons why things went differently this time, starting with the fact that Linus took an interest in the problem. But it's also true that performance-oriented patches really need to come with numbers to show that they are achieving the desired effect; had Andi taken the time to quantify the impact of his change, he would have had a stronger case for merging it.
