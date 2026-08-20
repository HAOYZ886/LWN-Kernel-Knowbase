---
title: Concurrent code and expensive instructions
url: https://lwn.net/Articles/423994/
date: "January 26, 2011"
category: Memory barriers
author: "January 26, 2011 This article was contributed by Paul McKenney"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

January 26, 2011

This article was contributed by Paul McKenney

#### Introduction

Symmetric multiprocessing (SMP) code often requires expensive instructions, including atomic operations and memory barriers, and often causes expensive cache misses. Yet some SMP code can be extremely cheap and fast, using no expensive instructions at all. Examples of cheap SMP code include per-CPU counters and RCU read-side critical sections. So why can't _all_ SMP code be cheap? Is it just that we aren't smart enough to spot clever ways of implementing other algorithms, for example, concurrent stacks and queues? Is it that we might be able to implement concurrent stacks and queues without expensive instructions, but only at the cost of mind-bending complexity? Or is it simply impossible to implement concurrent stacks and queues without using expensive instructions? 

My traditional approach has been to place my faith in two observations: (1) if you beat your head against a wall long enough, one of two things is bound to happen, and (2) I have a hard head. Although this approach has worked well, something less painful would be quite welcome. And so it was with great interest that I read a paper entitled "[Laws of Order: Expensive Synchronization in Concurrent Algorithms Cannot be Eliminated](<http://infoscience.epfl.ch/record/161286>)" by Attiya et al., with the "et al." including Maged Michael, whom I have had the privilege of working with for quite some time. 

It is important to note that the title overstates the paper's case somewhat. Yes, the paper does present some laws requiring that many concurrent algorithms use expensive instructions, however, all laws have their loopholes, including the Laws of Order. So while we do need to understand the Laws of Order, we most especially need to understand how to fully exploit their loopholes. 

To arrive at the Laws of Order, this paper first expands the definition of commutativity to include sequential composition, which in the C language can best be thought of as the ";" operator. In this case, commutativity depends not just on the operator, but on the operands, which for our purposes can be thought of as calls to arbitrary C functions. For example, the statements: 

```
atomic_inc(&x); 
        atomic_inc(&y);
```

are commutative: the values of `x` and `y` are the same regardless of the order of execution. In contrast: 

```
atomic_set(&x, 1); 
        atomic_set(&x, 2);
```

are non-commutative: the value of `x` will be either 1 or 2, depending on which executes first. 

These examples execute sequentially, but the paper considers concurrent execution. To see this, consider a concurrent set that has these operations: 

  * a set-member-addition function (call it `set_add()`) that returns an indication of whether the element to be added was already in the set, 

  * a set-member-test function (call it `set_member()`), and 

  * a set-member-removal function (call it `set_remove()`) that returns an indication of whether anything had actually been removed. 

Then concurrently testing two distinct members is commutative: the order in which `set_member(&s, 1)` and `set_member(&s, 2)` execute will not affect the return values from either, and the final value of set `s` will be the same in either case. Therefore, it is not necessary for the two invocations to coordinate with each other. The fact that coordination is not required means that there is some hope that expensive instructions are not needed to implement `set_member()`. 

In contrast, concurrent invocation of `set_add(&s, 1)` and `set_member(&s, 1)` would _not_ be commutative if the set `s` initially did not contain the value 1\. The `set_member()` invocation would return true only if it executed after the `set_add()`. Some coordination between the two functions is clearly required. 

The most important results of the paper rely on a strong form of non-commutativity, which is strangely enough termed "strong non-commutativity", which we can abbreviate to "SNC". The example in the previous paragraph is not SNC because, while `set_add()` can affect `set_member()`, the reverse is not the case. In contrast, an SNC pair of functions would each affect the other's result. For example, consider `set_add(&s, 1)` and `set_remove(&s, 1)`, where the set `s` is initially empty. If `set_add(&s, 1)` executes first, then both functions will indicate success, and set `s` will be empty. On the other hand, if `set_remove(&s, 1)` executes first, then only the `set_add(&s, 1)` will indicate success and the set `s` will contain 1\. In this case, the return value of `set_remove()` is affected by the order of execution. On the other hand, if the set `s` initially contains 1, it will be `set_add(&s, 1)` whose return value is affected. Therefore, the order of execution can affect the return value of both functions, and these functions are therefore SNC. 

**Quick Quiz 1** : Is `atomic_add_return()` SNC? In other words, are multiple concurrent calls to this function SNC?   
Answer

The key result of the paper is that under certain conditions, the implementation of a pair of SNC functions must contain a heavyweight instruction, where a "heavyweight instruction" can either be an atomic read-modify-write instruction or a heavyweight memory barrier. In the Linux kernel, only `smp_mb()` qualifies as a heavyweight memory barrier. 

The "certain conditions" are: 

  1. Both functions in the pair must be [deterministic](<http://en.wikipedia.org/wiki/Deterministic_algorithm>), in other words, the final state (including return values) must be a strict function of the initial state and order of execution. 

  2. The functions must be [linearizable](<http://en.wikipedia.org/wiki/Linearizability>). 
Interestingly enough, although the paper requires that the implementation of an SNC, deterministic, and linearizable pair of functions each contain at least one heavyweight instruction, it does _not_ require that this instruction be executed on each invocation. 

**Quick Quiz 2** : Imagine an increment function that is not permitted to lose counts even when multiple invocations execute concurrently, and that does not return the value of the counter. Must the implementation of such a function contain an atomic read-modify-write instruction or a heavyweight memory barrier?   
Answer

So if we want our code to run fast, we have four ways to avoid heavyweight instructions: 

  1. Formulate the API to be non-SNC. 

  2. Design the implementation so that any required heavyweight instructions almost never need to actually be executed. 

  3. Accept non-determinism. 

  4. Accept non-linearizability. The paper ignores this possibility, possibly due to the common academic view that non-linearizable algorithms are by definition faulty. 

Interestingly enough, [relativistic programming](<http://wiki.cs.pdx.edu/rp/>) has long suggested use of several of these approaches to attain good performance and scalability. The "Laws of Order" therefore provides a good theoretical basis for understanding why relativistic programming is both desirable and necessary. 

Let's take a look at some examples, starting with a memory allocator. Given that concurrent calls to `kmalloc()` are not supposed to return a pointer to the same block of memory, we have to conclude that `kmalloc()` is SNC and thus might need heavyweight instructions in its implementation. 

**Quick Quiz 3** : How can we avoid the use of heavyweight instructions in the implementation of `kmalloc`? If it turns out to be impossible to completely avoid their use, how can we reduce the frequency of their execution?   
Answer

The second example is of course RCU. Let's focus on the `rcu_read_lock()`, `rcu_read_unlock()`, `synchronize_rcu()`, `rcu_dereference()` and `rcu_assign_pointer()` API members. The `rcu_read_lock()` function is unaffected by any of the other members, so any pair that includes `rcu_read_lock()` is non-SNC, which is why this function need not include any heavyweight instructions. The same is true of `rcu_read_unlock()`. 

Interestingly enough, `synchronize_rcu()` is affected by both `rcu_read_lock()` and `rcu_read_unlock()`, in that the former can prevent `synchronize_rcu()` from returning and the latter can enable it to return. However, neither `rcu_read_lock()` nor `rcu_read_unlock()` is affected by `synchronize_rcu()`. This means that `synchronize_rcu()` is non-SNC and might therefore have an implementation that does not use heavyweight instructions. However, such an implementation seems quite implausible if you include the actions of the updater both before and after the call to `synchronize_rcu()` in conjunction with the RCU read-side critical section. The paper, though, considers only data flowing via those function's arguments and return value. It would be interesting to see a generalization of this work that includes side effects. 

My guess is that for a given code fragment to be non-SNC, any conceivable API would need to be non-SNC. If my guess is correct, then the full RCU update is non-SNC with respect to any RCU read-side critical section containing `rcu_dereference()`. The reasoning is that the return value from `rcu_dereference()` can be affected by the RCU update, and the duration for which `synchronize_rcu()` blocks can be affected by `rcu_read_lock()` and `rcu_read_unlock()`. 

**Quick Quiz 4** : Are there any conditions in which `rcu_read_unlock()` will be SNC with respect to `synchronize_rcu()`?   
Answer

Finally, let us look at the set implementation that includes `set_add()`, `set_member()`, and `set_remove()`. We saw that `set_add()` and `set_remove()` were SNC. 

**Quick Quiz 5** : Is there any way to implement `set_add()` and `set_remove()` without using heavyweight instructions?   
Answer

Of course, this paper does have a few shortcomings, many of which fall under the rubric of "future work": 

  1. The paper describes the theoretical limitations at great length, but does not describe many ways of avoiding them. However, I am quite confident that the Linux kernel community will be more than able to produce good software engineering solutions that work around these limitations. In fact, there is a lot to be said for letting the theoreticians worry about limitations and letting us hackers worry about solving problems in spite of those limitations.

  2. The paper focuses almost exclusively on reordering carried out by the CPU. It turns out that reordering due to compiler optimizations can be at least as "interesting" as CPU reordering. These sorts of compiler optimizations are allowed by the current C-language standard, which permits the [compiler to assume that there is only one thread in the address space.](<http://www.hpl.hp.com/techreports/2004/HPL-2004-209.html>) Within the Linux kernel, the `barrier()` directive restricts the compiler's ability to move code, and this directive (or its open-coded equivalent) is used in locking primitives, atomic operations, and memory barriers.

  3. There is some uncertainty about exactly what properties of code must be SNC for this paper's results to hold. The paper focuses almost exclusively on function arguments and return values, but my guess is that the list of properties is quite general. For example, an unconditional lock-acquisition primitive certainly seems like it should be covered by this paper's result, but such primitives do not return a value. Can the fact that the second of two concurrent acquisitions simply fails to return be considered to be evidence of the SNC nature of lock acquisition? If not, exactly why not? If so, exactly what is the set of effects that must be taken into account when judging whether or not this code fragment is SNC?

This seems to be a future-work topic.

  4. A bit of thought about the results of this paper give clear reasons why it is [often so hard to parallelize existing sequential code](<http://paulmck.livejournal.com/16478.html>). Sequential code inflicts no penalties for the use of SNC APIs, so SNC APIs can be expected to appear in sequential code even when a non-SNC API might have served just as well. After all, what programmer could resist the temptation to make `set_add()` return an indication of whether the element was already in the set? The paper would have done well to state this point clearly.

  5. The paper fails to call out non-linearizability as a valid loophole to its laws of order.

  6. An interesting open question: What are the consequences of using one of the loopholes of the laws of order? In my limited personal experience, leveraging non-linearizability and privatization permits full generality (for example, parallel memory allocators), while leveraging non-SNC and non-determinism results in specialized algorithms (for example, RCU). It would be quite interesting to better understand any theoretical and software-engineering limitations imposed by these loopholes.

  7. The paper overreaches a bit when it states that: 

For synthesis and verification of concurrent algorithms, our result is potentially useful in the sense that a synthesizer or a verifier need not generate or attempt to verify algorithms that do not use RAW [`smp_mb()`] and AWAR [atomic read-modify-write operations] for they are certainly incorrect. 

As we have seen, it is perfectly legal for a concurrent algorithm to avoid use of these operations as long as that algorithm is either: (1) non-SNC, (2) non-deterministic, or (3) non-linearizable. There are a few other places where the limitations on the main result are not stated as carefully as they should be. Given that the rest of the paper seems quite accurate and on-point, I would guess that this sentence is simply an honest error that slipped through the peer-review process. We all make mistakes. 

Although I hope that these shortcomings will be addressed, I hasten to add that they are insignificant compared to the huge step forward that this paper represents. 

In summary, the "Laws of Order" paper shines some much-needed light on the question of whether heavyweight instructions are needed to implement a given concurrent algorithm. Although I am not going to say that this paper fully captures my parallel-programming intuition, I am quite happy that it does land within a timezone or two, which represents a great improvement over previous academic papers. But the really good news is that the limitations called out in this paper have some interesting loopholes that can be exploited in many cases. If the Linux kernel community pays careful attention to both the limitations and the loopholes called out in this paper, I am confident that the community's already-impressive parallel-programming capabilities will become even more formidable. 

####  Acknowledgments

I owe thanks to Maged Michael, Josh Triplett, and Jon Walpole for illuminating discussions and for their review of this paper, and to Jim Wasko for his support of this effort. 

####  Legal Statement

This work represents the view of the author and does not necessarily represent the view of IBM. 

Linux is a registered trademark of Linus Torvalds. 

Other company, product, and service names may be trademarks or service marks of others. 

####  Answers to Quick Quizzes

**Quick Quiz 1** : Is `atomic_add_return()` SNC? In other words, are multiple concurrent calls to this function SNC? 

**Answer** : Yes. Suppose that an `atomic_t` variable named `a` is initially zero and that a pair of concurrent `atomic_add_return(1, &a)` functions execute. The first one to execute will return zero, and the second one will return one. Each instance's return value is therefore affected by the order of execution, which indicates strong non-commutativity. 

This may seem strange, given that addition is commutative. And in fact the final value of `a` will be two regardless of order of execution. 

To see the reasoning behind the definition of SNC, consider `atomic_inc(&a)`, which also adds one to `a` but does not return the initial value. In this case, because there are no return values, the invocations of `atomic_inc(&a)` cannot possibly affect each others' return values. 

Therefore, `atomic_inc(&a)` is non-SNC. 

It is interesting to note that the designers of the Linux kernel's suite of atomic operations had an intuitive understanding of the results of this paper. The atomic operations that return a value (and thus are more likely to be SNC) are the ones that are required to provide full memory ordering. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : Imagine an increment function that is not permitted to lose counts even when multiple invocations execute concurrently, and that does not return the value of the counter. Must the implementation of such a function contain an atomic read-modify-write instruction or a heavyweight memory barrier? 

**Answer** : No. Although this function is deterministic and linearizable, it is non-SNC. And in fact such a function could be implemented via a "split counter" that uses per-CPU non-atomic variables. Because each CPU increments only its own variable, counts are never lost. To get the aggregate value of the counter, simply sum up the individual per-CPU variables. 

Of course, it might be necessary to disable preemption and/or interrupts across the increments, but such disabling requires neither atomic read-modify-write instructions nor heavyweight memory barriers. 

However, the linearizability of this function depends on the counter always being incremented by the value 1. To see this, imagine a counter with an initial value of zero to which three CPUs are concurrently adding the values 3, 5, and 7, and that meanwhile three other CPUs are reading out the counter's value. Because there are no ordering guarantees, these three other CPUs might see the additions in any order. One of these CPUs might add the per-CPU variables and obtain a sum of 3, another might obtain a sum of 5, and the third might obtain a sum of 7. These three results are not consistent with any possible ordering of the additions, so this counter is not linearizable. 

However, for a great many uses, this lack of linearizability is not a problem. 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : How can we avoid the use of heavyweight instructions in the implementation of `kmalloc()`? If it turns out to be impossible to completely avoid their use, how can we reduce the frequency of their execution? 

**Answer** : The usual approach is to observe that a given pair of invocations of `kmalloc()` invocations will be SNC only if it is possible for them to be satisfied by the same block of memory. The usual way to greatly reduce the probability of a pair of `kmalloc()` invocations fighting over the same block of memory is to maintain per-CPU pools of memory blocks, which is what the Linux kernel's implementation of `kmalloc()` actually does. Heavyweight instructions are executed only if a given CPU's pool either becomes exhausted or overflows. This approach is related to the paper's suggestion of using "single-owner" algorithms. 

It might be possible to avoid heavyweight instructions by introducing non-determinism, for example, by making `kmalloc()` randomly fail. This can certainly be accomplished by making `kmalloc()` unconditionally return `NULL` if the CPU's pool was exhausted, but such an implementation might not prove to be fully satisfactory to its users. Coming up with a reasonable implementation that uses non-determinism to avoid heavyweight instructions is left as an exercise for the adventurous reader. 

Similarly, eliminating heavyweight instructions by introducing non-linearizability is left as an exercise for the adventurous reader. 

**Back to Quick Quiz 3**.

**Quick Quiz 4** : Are there any conditions in which `rcu_read_unlock()` will be SNC with respect to `synchronize_rcu()`? 

**Answer** : In some implementations of RCU, `synchronize_rcu()` can interact directly with `rcu_read_unlock()` when the grace period has extended too long, either via [`force_quiescent_state()`](<http://lwn.net/Articles/305782/>) machinations our via [RCU priority boosting](<http://lwn.net/Articles/220677/>). In these implementations, `rcu_read_unlock()` will be SNC with respect to `synchronize_rcu()`. The Linux kernel's Preemptible Tree RCU is an example of such an implementation, as can be seen by examining the `rcu_read_unlock_special()` function in `kernel/rcutree_plugin.h`. This code executes rarely, thus using the second loophole called out above ("Design the implementation so that any required heavyweight instructions almost never need to actually be executed"). 

**Back to Quick Quiz 4**.

**Quick Quiz 5** : Is there any way to implement `set_add()` and `set_remove()` without using heavyweight instructions? 

**Answer** : This can be done easily for sets containing small integers if there is no linearizability requirement. The set is represented as a dense array of bytes so that each potential member of the set maps to a specific byte. The `set_add()` function would set the corresponding byte to one, the `set_remove()` function would clear the corresponding byte to zero, and the `set_member()` function would test the corresponding byte for non-zero. 

This implementation is non-linearizable because different CPUs might well disagree on the order that members were added to and removed from the set. 

**Back to Quick Quiz 5**.
