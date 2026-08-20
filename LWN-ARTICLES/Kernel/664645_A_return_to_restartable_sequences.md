---
title: A return to restartable sequences
url: https://lwn.net/Articles/664645/
date: "November 18, 2015"
category: Restartable sequences
author: "By Jonathan Corbet November 18, 2015"
---

By **Jonathan Corbet**  
November 18, 2015

Once upon a time, highly concurrent programming was something that only a small subset of kernel developers needed to worry about. As the number of cores found in a CPU package grows, though, concurrency concerns are moving out to user space. The concerns of interest here are not just the protection of critical sections; user space has had to be able to do that for many years. A different level of worry comes to the fore at levels of concurrency where the overhead of locking becomes a significant performance issue in its own right. That's when developers start to think about lockless algorithms, which bring unique challenges in user space. 

In the kernel, lockless programming tends to be tricky, leading to code that can be brittle if the data-access rules are not well understood or observed. But kernel code has a distinct advantage over user-space code in this regard: it is able to ensure that critical-section code can run to completion without being preempted. As long as the code restricts itself to per-CPU data structures, running with preemption disabled guarantees that no other thread will try to access those structures concurrently. User-space code has no such luxury; it always runs with preemption enabled. So any attempt to use per-CPU data structures in a lockless mode must use a different approach. 

One such approach has been termed "restartable sequences"; the first patch enabling restartable sequences was [examined here](<https://lwn.net/Articles/650333/>) last July. A [new patch set](<https://lwn.net/Articles/662228/>) was posted alongside the [kernel-summit session](<https://lwn.net/Articles/662946/>) on restartable sequences in October. This patch features a different implementation and API that should address a number of the worries raised by the first attempt. 

A restartable sequence is a brief segment of code performing some sort of lockless operation on a per-CPU data structure. A key rule is that the visible effects of a restartable sequence must be made by a single instruction at the very end of the sequence. Imagine, for example, the following (simplistic) code removing the head item from a linked list: 

```
struct list_thingie *item, *new_head;
    
        item = percpu_thingie_list_head;
        new_head = item->next;
        percpu_thingie_list_head = new_head;
```

The final line is the only operation that would be visible to other threads running on the same CPU. The operation could be interrupted anytime before that assignment without any ill effects — assuming the interrupted thread did not actually try to use `item`, of course. That final line could also be implemented as a single instruction. So this little fragment of code could meet the rules for a restartable sequence; properly implemented, it could allow multiple threads to remove items from a shared list without the need for locking. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There is one other thing that is needed, though, for a proper restartable sequence: some code to execute if the sequence happens to be interrupted partway through. In most cases (this one included), that code simply needs to restart the sequence from the beginning. With that in place, a restartable sequence can safely run in a lockless mode, but only if it can either (1) run to completion, or (2) know that it has been interrupted and jump to the failure code. That is where the need for kernel support comes in. 

In the new patch, an application wanting to use restartable sequences needs to register two addresses with the kernel, using a new system call: 

```
int restartable_sequences(int flags, unsigned long *counter, void *post_commit);
```

(`restartable_sequences()` is the name used in the implementation; the associated [test code](<https://lwn.net/Articles/664654/>) calls it `rseq()`, though). 

The `flags` argument is currently unused. The `counter` address is a location where the kernel stores a combination of the current CPU number and the current "event counter" — the number of times the thread has been preempted. The application should initially store NULL in the location pointed to by `post_commit`; later, when a restartable sequence is active, the application should store the address of the first instruction following the commit instruction there. Note that this call does not actually start a restartable sequence; instead, it sets up the infrastructure so that such sequences can be run. 

To actually run a restartable sequence, the application thread must carefully do the following things, in order: 

  1. Read the current CPU/event counter value from the `counter` address provided to the kernel above; this value must be stored for future use. 

  2. Place the address to jump to should the sequence fail (i.e. if the thread is preempted while the sequence is running) where the kernel will find it. The actual location for this address is architecture-dependent; the x86_64 implementation wants it in the CX processor register. 

  3. Load the address of the first post-commit instruction into the `post_commit` address provided above. 

  4. Check the `counter` value again and ensure that it matches the value stored in the first step. If the two do not match, preemption has already occurred and the code should jump directly to the failure address 

  5. Execute the critical section through the final commit instruction. 

  6. Clear the post-commit instruction address stored in step 3. 

The kernel's test for whether a restartable sequence is active is simple: is the current instruction pointer less than the address of the post-commit instruction stored in step 3? It is thus the storing of that address that begins the sequence for real; once that happens, the kernel will cause the thread to jump to its failure address if it is preempted. The manual check in step 4 is needed, though, in case preemption happened just before the execution of step 3\. Performing the steps in this order ensures that there are no race conditions around the preemption checking. 

In the previous version of the patch, the entire restartable sequence almost certainly needed to be written in assembly. This new interface does not eliminate the need for assembly code, but it does reduce the amount of that code considerably. A few instructions around and including the final commit must still be done in assembly, though that can probably be hidden in library code for a number of common use cases. 

The previous version of the patch required the registration of one memory area that would hold critical-section code. With this version, instead, the critical section(s) can appear anywhere. Library code could _almost_ use this feature independently of other application code, with one exception: the two addresses passed to the `restartable_sequences()` call must be shared by all users. The alternative would be to make a new `restartable_sequences()` call prior to beginning each sequence, but that is likely to run fairly strongly counter to the performance objectives that motivated the use of restartable sequences in the first place. 

Discussion of this version of the patch set has been muted; perhaps it got lost in all the other kernel summit activity. Interest in this feature clearly goes beyond Google (where the patch originates), though. One would thus expect this feature to eventually enter the kernel in some form. Some of the implementation concerns from last time around have been addressed; the impact on the scheduler has been reduced, for example. Whether it will take another iteration or two to get the user-space interface right remains to be seen, though.
