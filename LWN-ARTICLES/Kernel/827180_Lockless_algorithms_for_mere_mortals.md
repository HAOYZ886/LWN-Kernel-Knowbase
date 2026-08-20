---
title: Lockless algorithms for mere mortals
url: https://lwn.net/Articles/827180/
date: "July 28, 2020"
category: Lockless algorithms; Scalability
author: "By Jonathan Corbet July 28, 2020"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 28, 2020

Time, as some have said, is nature's way of keeping everything from happening at once. In today's highly concurrent computers, though, time turns out not to be enough to keep events in order; that task falls to an extensive set of locking primitives and, below those, the formalized view of memory known as the Linux kernel memory model. It takes a special kind of mind to really understand the memory model, though; kernel developers lacking that particular superpower are likely to make mistakes when working in areas where the memory model comes into play. Working at that level is increasingly necessary for performance purposes, though; a recent conversation points out ways in which the kernel could make that kind of work easier for ordinary kernel developers. 

Concurrency comes into play when multiple threads of execution are accessing the same data at the same time. Even in a simple world, keeping everything coherent in a situation like this can be a challenging task. The kernel prevents the wrong things from happening at the same time with the use of spinlocks, mutexes, and other locking primitives that can control concurrency. Locks at this level can be thought of as being similar to traffic lights in cities: they prevent accidents as long as they are properly observed, but at the cost of stopping a lot of traffic. Time spent waiting for locks hurts; even the time bouncing lock data between memory caches can wreck scalability, so developers often look for ways to avoid locking. 

#### Lockless list linking

Consider a highly simplified example: inserting an element into a singly-linked list. One possible solution would be to use a mutex to protect the entire list; any thread traversing the list must first acquire this mutex. If the thread inserting an element acquires this lock, it knows that no other thread will be traversing the list at the same time, so changes will be safe. But if this list is heavily used, this locking will quickly become expensive. 

So one might consider a lockless alternative. If the list initially looks like this: 

> ![\[linked list\]](https://static.lwn.net/images/conf/2013/lce-ks/rcu-ll1.png)

One could start by linking the outgoing pointer from the new element to the existing element that will soon follow it in the list: 

> ![\[linked list\]](https://static.lwn.net/images/conf/2013/lce-ks/rcu-ll2.png)

At this point, the list still looks the same to any other thread that is traversing it. To complete the operation, the code will redirect the pointer from the preceding element in the list to the new element: 

> ![\[linked list\]](https://static.lwn.net/images/conf/2013/lce-ks/rcu-ll3.png)

Now everybody sees the new list; no locking was required, and the view of the list was always consistent and correct. Or so one would hope. 

The problem is that modern hardware makes things harder in the name of performance. The order in which a series of operations is executed may not be the order in which those operations are visible to other threads in the system. So, for example, it might well be that other threads in the system see the assignment of the two pointers above in the opposite order, with the result that, from their point of view, there is a window of time during which the list looks like this: 

> ![\[broken linked list\]](https://static.lwn.net/images/2020/ll-broke.png)

The outgoing pointer in the new element will contain whatever was there before the assignment happened, leading to a list that heads off into the weeds. There are few certainties in the world, but one can be reasonably confident in saying that little good will come from this situation. 

#### A more complex view of memory

Another way to think about it is that locking provides a sort of Newtonian-physics view of the world; in a given situation, one always knows what's going to happen. At lower levels, life starts to resemble quantum physics, where surprising things can happen and few people can convincingly claim to understand it all. One can normally function quite well in this world without understanding quantum physics, but there are situations where it is good to have an expert around. 

The Linux kernel has a few experts of this type; they have put a great deal of work over the years into the creation of the kernel's memory model, which is a description of how the kernel views memory and the how to safely perform operations where concurrency may come into play. The result is the infamous [memory-barriers.txt](<https://www.kernel.org/doc/Documentation/memory-barriers.txt>) documentation file and a whole raft of supporting materials. From this documentation, one can learn a couple of useful things for the list-insertion example given above: 

  * Code traversing the list should read the next-item pointers with an "acquire" operation, which is a special sort of barrier. It guarantees that any operation that happens after the barrier actually appears afterward elsewhere in the system. In this case, it would ensure that, if a traversal reads a pointer to a given element, the assignment of that element's "next" pointer will already be visible. 
  * Code that sets the pointer should do so with a "release" operation, which ensures that any other operations done before the release operation are visible before that operation. That will ensure that a new element's "next" pointer is seen correctly globally before the pointer to the element itself. 

This example is complicated enough to explain, but it is as simple as these things get; most cases are rather more complex. To make things worse, optimizing compilers can [create surprises of their own](<https://lwn.net/Articles/799218/>) in pursuit of higher performance. The kernel's memory model strives to address this threat as well. 

#### The problem with the memory model

Recently, Eric Biggers posted [a patch](<https://lwn.net/ml/linux-fsdevel/20200713033330.205104-1-ebiggers@kernel.org/>) fixing a perceived problem in the direct I/O code where a concurrent data-access situation lacked the appropriate barriers. There was some discussion about whether a bug actually existed or not; the problem according to Biggers is that this sort of concurrent access is deemed "undefined behavior", meaning that the compiler is granted license to pursue any evil agenda that might strike its fancy. The real dispute, though, was over the fix. 

Dave Chinner, who is generally acknowledged as being a moderately competent kernel developer, [complained](<https://lwn.net/ml/linux-fsdevel/20200716014656.GJ2005@dread.disaster.area/>) that the resulting code was not something that could be readily understood: 

I'm talking from self interest here: I need to be able to understand and debug this code, and if I struggle to understand what the memory ordering relationship is and have to work it out from first principles every time I have to look at the code, then *that is bad code*. 

He was pointed to the memory-model documentation, but that did little to improve his view of the situation: 

The majority of the _good_ programmers I know run away screaming from this stuff. As was said many, many years ago - understanding memory-barriers.txt is an -extremely high bar- to set as a basic requirement for being a kernel developer. 

This documentation, he said, is aimed at people who spend their time thinking about memory-ordering issues. Everybody else is going to struggle with it, starting with the basic terminology used; even those who manage to understand it are likely to forget again once they go back to the problems they are actually trying to solve. Kernel developers would be better served, Chinner said, with a set of simple recipes showing how to safely code specific lockless patterns. 

Biggers responded by posting [a documented recipe](<https://lwn.net/ml/linux-kernel/20200717044427.68747-1-ebiggers@kernel.org/>) for the "initialize once" pattern that was the source of the original problem in the direct-I/O subsystem. This pattern comes about when the initialization of a data structure is deferred until the structure is actually used, perhaps because that use may never actually happen. The initialization should be done exactly once; two racing threads should not both try to carry it out. The document provided several recipes of increasing complexity intended to match different performance needs. 

While the attempt to provide useful recipes was welcomed, it became clear that a number of people felt that the effort had missed the mark somewhat. Darrick Wong, for example, [pointed out](<https://lwn.net/ml/linux-kernel/20200717205340.GR7625@magnolia/>) that language like: 

Specifically, if all initialized memory is transitively reachable from the pointer itself, then there is no control dependency so the data dependency barrier provided by READ_ONCE() is sufficient. 

is not immediately clear to a lot of developers. Alan Stern's [attempt to clarify it](<https://lwn.net/ml/linux-kernel/20200718012555.GA1168834@rowland.harvard.edu/>) read like this: 

Specifically, if the only way to reach the initialized memory involves dereferencing the pointer itself then READ_ONCE() is sufficient. This is because there will be an address dependency between reading the pointer and accessing the memory, which will ensure proper ordering. But if some of the initialized memory is reachable some other way (for example, if it is global or static data) then there need not be an address dependency, merely a control dependency (checking whether the pointer is non-NULL). Control dependencies do not always ensure ordering \-- certainly not for reads, and depending on the compiler, possibly not for some writes -- and therefore a load-acquire is necessary. 

This was seen as driving home the point that started the whole discussion: most developers do not think of memory this way and would really rather not have to. They simply do not think in this kind of language. As long as lockless algorithms require this sort of understanding, they will be underused and many of the implementations that do show up are likely to be buggy in subtle ways. 

An alternative, as [suggested](<https://lwn.net/ml/linux-kernel/20200717174750.GQ12769@casper.infradead.org/>) by Matthew Wilcox, is to define an API for this sort of on-the-fly initialization and hide the details behind it. Developers who understand memory models and enjoy thinking about them can concern themselves with optimizing the implementation, while the rest of the kernel-development community can simply use it with the knowledge that it works as intended. There followed a discussion with several different ideas about how this API should actually look, but nothing emerged that looks like it could find its way into wider use. 

This particular discussion has faded away, but the underlying problem remains. Successful software development at all levels depends on the management of complexity, but that is not yet really happening at the memory-model level. Sooner or later, somebody will come along with the right skills to both understand the Linux kernel memory model and to hide it behind a set of APIs that other developers can safely use without having to understand that model. Until then, writing lockless code will continue to be a challenging task for many developers — and the people who have to review their code.
