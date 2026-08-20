---
title: Faster RCUs and lockless memory allocation
url: https://lwn.net/Articles/1081009/
date: "July 7, 2026"
category: "Read-copy-update"
author: "By Daroc Alden July 7, 2026 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
July 7, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Puranjay Mohan shared some of the [ work](<https://lwn.net/ml/all/20260417231203.785172-1-puranjay@kernel.org/>) he's been doing recently on improving the performance of read-copy-update (RCU) at the 2026 [ Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>); his talk would have been nice context to have earlier in the day when Harry Yoo and Alexei Starovoitov led a session about the [ new `kmalloc_nolock()` function](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=af92793e52c3>) that allows for lockless allocation from any kernel context, and which interacts with the RCU subsystem to allow that. This article therefore covers the two sessions together and in the reverse order, to provide that missing context. 

#### Faster RCU periods

The idea for Mohan's performance work began, as so many RCU-related ideas do, while chatting with Paul McKenney. RCU works to protect data behind a pointer by having writers create a new copy with any changes they wish to make, and then atomically swapping the pointer to point to the new copy. Readers may still be reading the old version, so the writer cannot free the old version until it can be sure that all of the readers have moved on. 

[ ![\[Puranjay Mohan\]](https://static.lwn.net/images/2026/puranjay-mohan-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1081033>)

That happens whenever the kernel reaches a "quiescent state": every CPU has gone through a context switch, idle, return to user space, or other transition that ensures no reader can still exist. The RCU subsystem counts these events. When the old version of an RCU-protected resource needs to be freed, the subsystem waits until the counter is high enough (two more than the value of the counter when the new copy was swapped in, so that at least one full period is guaranteed to have elapsed) before freeing the memory. This wait is called the RCU grace period. The work that is waiting on a grace period is managed using a linked list of callbacks. 

But there are two different kinds of grace period. A call to [ `synchronize_rcu()`](<https://elixir.bootlin.com/linux/v7.1.2/source/kernel/rcu/tree.c#L3311>) passively waits for CPUs to report that they have reached a quiescent state, which can take tens of milliseconds. A call to [ `synchronize_rcu_expedited()`](<https://elixir.bootlin.com/linux/v7.1.2/source/kernel/rcu/tree_exp.h#L904>) sends inter-processor interrupts (IPIs) to force CPUs to reach and report quiescent states faster than normal. The second kind of grace period is, naturally, used in places where the kernel does not want to wait as long. 

The problem is that these mechanisms run concurrently and don't know about each other. A callback waiting for the end of a normal RCU grace period is not necessarily run when an expedited grace period ends, and vice versa. The idea that came up in Mohan's discussion with McKenney is to fix this — allow normal RCU callbacks to be executed as soon as an expedited grace period ends. Expedited grace periods happen frequently in some workloads, especially when the system is under memory pressure, so this change could result in RCU-protected resources being freed much more quickly. 

The fix is conceptually simple: have the callback list track both the non-expedited grace-period number and the expedited grace-period number, and consider callbacks eligible to run as soon as either is high enough. Alas, the details are a bit more complicated, especially because RCU is a performance-critical part of the kernel. There were three internal kernel functions — `rcu_exp_wait_wake()`, `rcu_pending()`, and `rcu_core()` — that needed adjustments to their logic to handle the change. 

Once it was working, Mohan ran a series of benchmarks to evaluate what the actual impact of the change would be. He set up fifteen threads to create and destroy sockets in a loop, which use RCU to free the memory. Other threads deliberately triggered expedited grace periods. Then he measured how the rate of expedited grace periods impacted the memory usage and latency of socket destruction. Overall, he saw 33%-41% less allocated memory (since memory was being freed earlier), across all tested expedited grace period rates. The latency of `synchronize_rcu()` calls also went down, although in a way that was more sensitive to the actual rate of expedited grace periods. 

Yoo asked whether it made sense to ask for an expedited grace period even when there is no particular need for one, just to reclaim memory. Mohan agreed that his patch set would make that possible, but Starovoitov objected that general best practice is to not use expedited grace periods unless they're really needed, since IPIs are expensive. During an out-of-memory situation, it is often too late for an expedited grace period to make a difference. Jakub Sitnicki suggested that an expedited grace period could be used as part of direct reclaim; Starovoitov directed people to ask McKenney and the memory-management folks about the consequences. 

Another member of the audience asked whether Mohan knew how often expedited grace periods were triggered in real workloads. Mohan didn't have production data for that, but expected it to vary significantly depending on the workload, and particularly on whether any BPF programs perform "map-in-map" updates, which consistently trigger expedited grace periods. That last question marked the end of the session. 

#### Lock-free memory allocation

In the past, programs that used BPF maps were required to preallocate the memory for those maps, Yoo explained as he opened the session on `kmalloc_nolock()`. This let the programs access the maps without needing to take the locks needed by the normal memory allocator, but it also wasted memory in the case that the program doesn't fill the map. The BPF allocator was [ added to the kernel](<https://lwn.net/Articles/883454/>) to allocate that memory on demand, but it introduced its own challenges. For one thing, inventing a new memory allocator is not fun. It introduces a maintenance burden that the BPF subsystem could do without. As originally designed, the BPF allocator also couldn't be used outside the BPF subsystem. The goal of the recent work on `kmalloc_nolock()` is to remove the BPF allocator without bringing back preallocation of BPF maps. 

The tricky part is that there are both sleepable and non-sleepable BPF programs that can access this memory. Non-sleepable programs can even access it from inside RCU critical sections. So, to provide a viable replacement for the BPF allocator, any solution needs to support allocating memory from both sleepable and non-sleepable contexts and potentially having that memory accessed from the other kind of context. It also needs to support having that memory be immediately freed, or even recycled using the [ typesafety-by-RCU mechanism](<https://www.kernel.org/doc/html/v6.16/RCU/rculist_nulls.html>). 

[ ![\[Harry Yoo\]](https://static.lwn.net/images/2026/harry-yoo-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1081035>)

After an RCU writer has swapped the pointer to an RCU-protected object, while it is waiting to be able to free the old version, there is the possibility that it will need to allocate another object of the same type. Normally, it would need to create a new allocation, since existing readers could still be referring to the old object. In some cases, however, it can be safe to reuse the existing allocation, even though readers may still be accessing it. Because using the allocation for a different type of object could lead to undefined behavior, this is only safe when any readers will expect the object to be of the exact same type. That it will be the same type, and not just an allocation of the same size, is guaranteed by the RCU subsystem, hence the name "typesafety by RCU". 

One place that this technique is used is in BPF hashmaps. In the kernel, almost all hashmaps use linked lists for hash buckets, where each element in the linked list contains a copy of the key and some associated data. The actual structure definition is more complicated, but it can be thought of as: 

```
struct node {
            long key;
            void *data;
            struct node *next;
        };
```

When removing an element from a hash bucket, the writer knows that any readers will need to check whether the key stored in the element is the one they're interested in. Therefore, after the object has been unlinked from the hash bucket, the writer can atomically change the key to something else, then update the data, before finally adding the object back to the appropriate bucket of the hashmap. Readers need to load the data pointer, followed by the key, and then validate that the key matches the object they wanted before using the data pointer. Since the writer updates the key first and the reader fetches the data pointer first, as long as appropriate memory ordering is used, the reader never reads incorrect data. They can read incorrect next pointers, however, which sends them traversing the wrong hash bucket. Therefore, each hash bucket ends with a structure saying which bucket this is the end of; if the reader reaches the end marker for a different bucket, it knows that it ran into this case, and needs to re-scan the correct bucket. This whole careful dance avoids needing to wait for the object to be returned to the kernel's general memory pool, reducing latency, memory churn, and memory usage. 

One audience member asked why the allocator would need to support memory being freed immediately. Starovoitov explained that some BPF iterators and helper functions can do that under some circumstances. Yoo's slides had included a mention of a `kfree_nolock()` to accompany `kmalloc_nolock()`. Amery Hung pointed out that it is already possible to use `kfree()` in these contexts, so there was probably no need for a separate `kfree_nolock()`. Yoo explained that the separate function is needed to support typesafety-by-RCU's caching, which is important for performance. 

Starovoitov elaborated by saying that ""back in the tracing days"" people would add elements to BPF maps and then remove them within nanoseconds. That was an especially common pattern for latency profiling — adding and removing a timestamp from a map. Freeing them using `free_rcu()` would go through a full RCU cycle, potentially much longer than the object had been needed in the first place. Using typesafety-by-RCU avoids that overhead, but the exact mechanism used to recycle freed memory doesn't really matter as long as there is some way to do this instant reuse, Starovoitov said. 

That prompted a digression about how this interacted with having different CPUs running BPF programs; in short, there's a potential for BPF programs to have race-condition-related correctness bugs here, but not in a way that impacts the kernel. 

Eventually, Yoo brought the topic back to `kmalloc_nolock()` and acknowledged that there would probably also need to be a `kfree_rcu_nolock()` to handle some cases. Starovoitov then asked about handling failure: `kmalloc_nolock()` succeeds ""pretty much all of the time"", but sometimes there is just no memory that is available without taking a lock. Yoo said that falling back to the buddy allocator and asking for a new slab might be a reasonable approach, but Starovoitov noted that that too can fail. He asked whether there was any way to handle failure in that case. 

Another round of discussion eventually produced a design that should be more robust to allocation failures, and support the ability to attach destructors to objects, but that was not able to completely avoid the possibility of allocation failure. Nevertheless, Starovoitov was pleased with the discussion, saying: ""I'm glad we talked it through."" The new design will, presumably, be a great improvement over the existing allocator all around.
