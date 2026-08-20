---
title: "C11 atomics part 2: \"consume\" semantics"
url: https://lwn.net/Articles/588300/
date: "February 26, 2014"
category: C11 atomic operations
author: "By Jonathan Corbet February 26, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 26, 2014

[Last week's article on C11 atomic variables](<https://lwn.net/Articles/586838/>) covered the discussion on the apparent mismatch between what the C11 standard defines and the kernel needs. This discussion did not conveniently end with the publication of the article, though. So this followup looks at the ground that was covered since then, with a particular focus on the "consume" memory order. It is possible, though far from guaranteed, that the outcome of this discussion could lead to changes in the standard to make it more applicable to kernel use. 

#### An introduction to consume

Much of the work around memory ordering relates to two modes of operation called "acquire" and "release". [This page](<http://en.cppreference.com/w/cpp/atomic/memory_order>) describes the meaning of these models within the standard. In short: a read from memory with "acquire" semantics is guaranteed to happen before any subsequent reads or writes in the same thread. A write with "release" semantics will happen (become globally visible) after any preceding reads or writes. The two are typically used together. Code that modifies a data structure will perform the final write (the one that makes any other data it wrote accessible globally) with a "release" operation, while code consuming that data will read the pointer to the data with an "acquire" operation. 

Acquire and release are useful concepts when trying to figure out how to work with shared data in a lockless manner. But in many cases an acquire operation provides stronger ordering than is really necessary. Reading with acquire semantics imposes ordering on _all_ subsequent reads and stores, even if many of those operations do not depend on the value that was read and could be more freely reordered by either the compiler or the processor. There is one case in particular in the kernel where it would be nice to have weaker (and cheaper) ordering guarantees than acquire provides. 

That case has to do with the read-copy-update (RCU) operation. The [LWN kernel index](<https://lwn.net/Kernel/Index/#Read-copy-update>) includes many articles on the details of RCU; to simplify things greatly here, for the purposes of this article it is hopefully enough to say that RCU works by putting potentially volatile data into structures that are accessed via pointers. Changing the data involves allocating a new structure, copying the new data into it, then updating the pointer to point to the new structure. Code consuming that data will see either the older or the newer pointer, depending on the relative timing of things, but either will be valid at the time. It is important, though, that the data written to the new structure all be globally visible before the pointer to that structure becomes visible; otherwise a consuming thread could end up reading the wrong information. 

This requirement can be met by assigning the pointer with release ordering, and reading the pointer (usually done with `rcu_dereference()`) with acquire ordering. But the only ordering that really matters is that between obtaining the pointer and accessing the contents of the structure it points to. On many processors, that ordering comes for free, with no expensive memory barriers required at all. 

Providing this weaker ordering is the role of the "consume" ordering, which only ensures that writes that are "dependent" on the read value must be visible. So in code that looks like this: 

```
p = rcu_dereference(pointer);
        q = p->something;
        *a = something_else;
```

With acquire ordering, the assignment to `*a` could not be reordered to happen before the `rcu_dereference()` call; with consume ordering, instead, that reordering could be done, and, on some architectures at least, the run-time cost of ensuring that ordering would be lower (or zero). Given that techniques like RCU are used in places where performance matters greatly, the extra performance obtained through the use of consume semantics seems worth having. 

#### Fixing consume

The problem with consume ordering as defined by the standard is that it requires extensive tracking of dependencies between data accesses. That tracking, it seems, is hard to understand and hard to do. The result is a standard text that is not entirely approachable to developers. There are also [reported bugs](<http://gcc.gnu.org/bugzilla/show_bug.cgi?id=59448>) in GCC indicating that the handling of consume ordering is not always done correctly. With some compilers, it seems, consume is just implemented as acquire, leading to correct results but losing the performance advantages that consume is supposed to provide. 

These problems make consume ordering sufficiently difficult to use in the kernel that, chances are, the kernel will continue to use its current mix of architecture-dependent macros and barriers. But what if the definition of consume ordering could be tweaked in the standard itself? There are (probably) few users of consume now, and many implementations likely just implement it as if it were acquire, so there may be scope for changes. 

Linus has [an idea](<https://lwn.net/Articles/588312/>) for a change that, he thinks, would solve most of the problems. He would like to get rid of the extensive language describing dependencies and their tracking and replace it with something simpler. His suggested wording is: 

The consume ordering guarantees the ordering between that atomic read and the accesses to the object that the pointer points to directly or indirectly through a chain of pointers. 

The idea here is simple and, in theory, it provides just the ordering that RCU needs. There are some interesting subtleties, though. The "chain of pointers" concept, for example, refers to assignments and simple modifications. So with an assignment like: 

```
p = rcu_dereference(something);
```

These assignments would create pointers in the chain: 

```
q = p;
        r = p + 1;
```

What the "chain" idea explicitly does not cover is aliases. If some other pointer in the function happens to point to the object that `p` points to, accesses to that object via the second pointer will not be ordered in any way. That makes Linus's idea of consume semantics different from that found in the standard; the latter requires the compiler to try to catch and handle that kind of aliasing. 

But what really makes up a "chain of pointers"? Paul McKenney, who would be the person who would have to try to sell any such concept to the standard committee, posted [a set of twelve rules](<https://lwn.net/Articles/588314/>) describing how these chains would be formed. There is an attempt to distinguish between pairs of operations like this (for example): 

```
q = p & ~0x1;
        r = p & 0x1;
```

This kind of logical AND operation is often found in the kernel; the lowest bits of pointers are sometimes used as flags to carry additional information about the pointer. The assignment of `q` above should preserve the dependency chain, while the assignment of `r` would not. Compilers can often detect assignments like that second one, which produces an integer value, not a pointer, and reorder them in surprising ways. 

It turns out that there are a lot of ways that one can destroy the essential "pointerness" of a pointer and break the dependency chain. In fact, there are so many that Linus [advised](<https://lwn.net/Articles/588315/>) (to put it politely) Paul to give up on trying to describe them: 

So *accept* the fact that some operations (and I guarantee that there are more of those than you can think of, and you can create them with various tricks using pretty much *any* feature in the C language) essentially take the data information away. And just accept the fact that then the ordering goes away too. 

An example he gave was: 

```
p = atomic_read(pp, consume);
        if (p == &variable)
            return p->val;
```

In this case, he said, the compiler could reasonably turn `p->val` into `variable.val`. At that point, there is no chain of pointers and no ordering; the read of `variable.val` could conceivably happen before the atomic read. If, instead, the `==` were to be changed to `!=`, the chain (and the ordering of the operations) would be preserved because there is no way for the compiler to know where `p` might point. 

#### Toward the standard?

After reading Linus's description, Paul [tried to write down the requirements](<https://lwn.net/Articles/588429/>) again, and came up with this summary: 

Therefore, the only operations that can be counted on to maintain the needed RCU orderings are those where the compiler really doesn't have any choice, in other words, where any reasonable way of computing the result will necessarily maintain the needed ordering. 

Linus more-or-less [agreed](<https://lwn.net/Articles/588430/>) that this is the case: 

So I think the C semantics should mirror what the hardware gives us \- and do so even in the face of reasonable optimizations - not try to do something else that requires compilers to treat "consume" very differently. 

He did go on to confess, though, that what he really wants is something like what Intel hardware provides. If he were the king of the world, he said, he would outlaw the weaker ordering provided by architectures like ARM and PowerPC. 

And that ties into [one of Paul's biggest concerns](<https://lwn.net/Articles/588432/>): will we be able to count on hardware providing the relatively strong Intel-style ordering in the future? Optimization techniques have advanced considerably over the years and will likely continue to do so. Paul wondered: ""Are ARM and Power really the bad boys here? Or are they instead playing the role of the canary in the coal mine?"" If the latter is true, then building a memory ordering regime around Intel's rules might prove hard to sustain over the long term. 

Responses so far suggest that others do not expect weaker ordering to hold in the long term; as [George Spelvin put it](<https://lwn.net/Articles/588434/>), once a processor adds the cache-coherency hardware to support other types of advanced optimization, it has the capability to provide stronger ordering anyway. Programming on systems with weaker memory ordering is harder, and thus more costly; there will come a time, some think, when those costs clearly are not justified if the ability to provide stronger ordering is available. 

Predicting the long-term future of computing hardware is hard, of course, and, meanwhile, systems with weaker ordering are around and must be supported. If Linus's model of consume semantics were to prevail, it could be supported on such hardware now with the use of appropriate memory barriers. But predicting whether Linus's vision might ever make it into a standard revision is just as hard. It might just have a persuasive champion who could present it to the committee in the proper language, but standard committees move in strange, mysterious, and slow ways. So this could be a story that plays out over years; in the meantime, the kernel will almost certainly not switch to C11 atomic variables for anything that benefits from consume-style semantics.
