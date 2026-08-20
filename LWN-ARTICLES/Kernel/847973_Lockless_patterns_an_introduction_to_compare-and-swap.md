---
title: "Lockless patterns: an introduction to compare-and-swap"
url: https://lwn.net/Articles/847973/
date: "March 12, 2021"
category: Lockless algorithms; cmpxchg
author: "March 12, 2021 This article was contributed by Paolo Bonzini Lockless patterns"
---

March 12, 2021

This article was contributed by Paolo Bonzini

* * *

[Lockless patterns](<https://lwn.net/Articles/844224/>)

In the first part of this series, I showed you the theory behind concurrent memory models and how that theory can be applied to simple loads and stores. However, loads and stores alone are not a practical tool for the building of higher-level synchronization primitives such as spinlocks, mutexes, and condition variables. Even though it is possible to synchronize two threads using the full memory-barrier pattern that was introduced last week ([Dekker's algorithm](<https://en.wikipedia.org/wiki/Dekker%27s_algorithm>)), modern processors provide a way that is easier, more generic, and faster--yes, all three of them—the _compare-and-swap_ operation. 

From the point of view of a Linux kernel programmer, compare-and-swap has the following prototype: 

```
T cmpxchg(T *ptr, T old, T new);
```

where `T` can be either an integer type that is at most as wide as a pointer, or a pointer type. In order to support such polymorphism, `cmpxchg()` is defined as a macro rather than a function, but the macro is written carefully to avoid evaluating its arguments multiple times. Linux also has a `cmpxchg64()` macro that takes 64-bit integers as the arguments, but it may not be available on all 32-bit platforms. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

`cmpxchg()` loads the value pointed to by `*ptr` and, if it is equal to `old`, it stores `new` in its place. Otherwise, no store happens. The value that was loaded is then returned, regardless of whether it matched `old` or not. The compare and the store are atomic: if the store is performed, you are guaranteed that no thread could sneak in and write a value other than `old` to `*ptr`. Because a single operation provides the old version of the value and stores a new one, compare-and-swap is said to be an _atomic read-modify-write_ operation. 

In Linux, the `cmpxchg()` macro puts strong ordering requirements on the surrounding code. A compare-and-swap operation comprises a load and a store; for the sake of this article, you can consider them to be, respectively, load-acquire and store-release operations. This means that `cmpxchg()` can synchronize with both load-acquire or store-release operations performed on the same location by other threads. 

#### Lock-free stacks and queues

"[Lockless algorithms for mere mortals](<https://lwn.net/Articles/827180/>)" already mentioned the use of compare-and-swap for lock-free lists. Here, we'll look at how a lockless, singly linked list could be implemented in C, and what it could be useful for. First of all, however, let's recap how a single-threaded C program would add an item in front of a singly-linked list: 

```
struct thing {
            struct thing *next;
            ...
        };
        struct thing *first;
    
        node->next = first;
        first = node;
```

Armed with the knowledge from the first part of the series, we know that we should turn the assignment to `first` into a store-release, so that `node->next` is visible to other threads doing a load-acquire. This would be an instance of the pattern presented there. 

However, that pattern only worked for a single producer and a single consumer; in the presence of multiple producers, the two instructions would have to be placed under a lock. This is because the value of `first` can change between the two instructions, for example if another element is added at the same time by another thread. If that happens, the outgoing pointer (`node->next`) in the new element will point to whatever `first` held before the assignment happened. This teaches us an important, if obvious, lesson: acquire and release semantics are just one part of designing and proving the correctness of lockless algorithms. Logic mistakes and race conditions can and will still happen. 

Instead of using a lock, `cmpxchg()` lets us catch the other thread in the act of modifying `first`. Something like this would work for any number of producers: 

```
if (cmpxchg(&first, node->next, node) == node->next)
            /* yay! */
        else
            /* now what? */
```

There are still a few things to sort out, as you can see. First and foremost, what to do if the `cmpxchg()` notices that `first` has changed. The answer in that case is simply to read the new value of `first` to `node->next` and try again. This is possible, because `node` is still invisible to other threads. Nobody will notice our stroke of bad luck. 

A second and more subtle question is: how do we load `first`? The load need not have either acquire or release semantics, because the code is not doing other memory accesses that depend on the value of `first`. On the other hand, perhaps the [big bad optimizing compiler](<https://lwn.net/Articles/793253/>) might think that `first` cannot change across iterations of the loop? Even though Linux's `cmpxchg()` does prevent this kind of compiler optimization, it is a good practice to mark relaxed loads and stores of shared memory using `READ_ONCE()` and `WRITE_ONCE()`. 

Putting everything together, we get: 

```
struct thing *old, *expected;
        old = READ_ONCE(first);
        do {
            node->next = expected = old;
            old = cmpxchg(&first, expected, node);
        } while (old != expected);
```

This is all nice, but it's only half of the story. We still have not seen how the list can be read on the consumer side. The answer is that it depends on the relationship between producers and consumers, the number of consumers, and whether the consumers are interested in accessing elements in LIFO (last-in-first-out) or FIFO (first-in-first-out) order. 

First of all, it could be that all reads happen after the producers have finished running. In this case, the synchronization between producers and consumers happens outside the code that manipulates the list, and the consumers can access the list through normal, non-atomic loads. The synchronization mechanism could be a thread-exit/thread-join pair such as the one we saw in the first article, for example. 

If reads are rare or can be batched, a more tricky implementation could allow producers to proceed locklessly, while reads would be serialized. Such an implementation could use a reader-writer lock (rwlock); however, the producers would take the lock for _shared_ access (with a `read_lock()` call) and the consumer(s) would take the lock for _exclusive_ access (with `write_lock()`)! This would also avoid reads executing concurrently with writes and, therefore, the consumer would be able to employ non-atomic loads. Hopefully, this example will show that there's no such thing as too many comments or too much documentation, even if you're sticking to the most common lockless programming patterns. 

If many consumers run concurrently with the producers, but they can consume the elements in any order, the consumers can obtain a whole batch of elements (removing them from the list) with a single instruction: 

```
my_things = xchg_acquire(&first, NULL);
```

`xchg()`, like `cmpxchg()`, performs an atomic combination of a read and a write to a memory location. In this case it returns the previous head of the list and writes `NULL` in its place, thus emptying the list. Here I am using the `xchg_acquire()` variant, which has acquire semantics for its load of `first`, but (just like `WRITE_ONCE()`) does not apply release semantics when it stores `NULL`. Acquire semantics suffice here, since this is still basically the same store-release/load-acquire pattern from part 1\. More precisely, it is a multi-producer, multi-consumer extension of that pattern. 

Should we do the same on the writer side and replace `cmpxchg()` with `cmpxchg_release()`? Indeed we could: in principle, all that the writer needs is to publish the store of `node->next` to the outside world. However, `cmpxchg()`'s acquire semantics when loading the list head have a useful side effect: they synchronize each writer with the thread that wrote the previous element. In the following picture, the load-acquire and store-release operations are all part of a successful series of `cmpxchg()` calls: 

```
thread 1: load-acquire first (returns NULL)
                  store-release node1 into first
                      \
          thread 2: load-acquire first (returns node1)
                    store-release node2 into first
                        \
             thread 3: load-acquire first (returns node2)
                       store-release node3 into first
                           \
                thread 4: xchg-acquire first (returns node3)
```

Thread 3's `cmpxchg()` is the only one to _synchronize with_ thread 4's `xchg_acquire()`. However, because of transitivity, _all_ `cmpxchg()`s _happen before_ the `xchg_acquire()`. Therefore, if `cmpxchg()` is used in the writers, the readers can go through the list with regular loads. 

If, instead, the writers used `cmpxchg_release()`, the happens-before relation would look like this: 

```
thread 1: load-acquire first (returns NULL)
                  store-release node1 into first
    
          thread 2: load first (returns node1)
                    store-release node2 into first
    
             thread 3: load first (returns node2)
                       store-release node3 into first
                           \
                thread 4: xchg-acquire first (returns node3)
```

Thread 4 would always read `node2` from `node3->next`, because it read the value that thread 3 wrote to `first`. However, there would be no _happens before_ edge from thread 1 and thread 2 to thread 4; therefore, thread 4 would need a `smp_load_acquire()` in order to see `node1` in `node2->next`. 

The above data structure is already implemented in Linux's [`linux/llist.h` header](<https://elixir.bootlin.com/linux/v5.11.2/source/include/linux/llist.h>). You're highly encouraged _not_ to reinvent the wheel and use that version, of course. That API, in fact, includes two more interesting functions: `llist_del_first()` and `llist_reverse_order()`. 

[`llist_del_first()`](<https://elixir.bootlin.com/linux/v5.11.2/source/lib/llist.c#L39>) returns the first element of the `llist` and advances the head pointer to the second element. Its documentation warns that it should only be used if there is a single reader. If, instead, there were two consumers, an intricate sequence of adds and deletes could lead to the so-called [ABA problem](<https://en.wikipedia.org/wiki/ABA_problem>). Since this article rests firmly on the principle of "if it hurts, don't do it", a detailed explanation is beyond its scope. However, it's worth pointing out the similarity with the earlier rwlock example. Just as in that case, multiple consumers will have to use locking to serialize concurrent access to the data structures. `llist_del_first()`, instead, lets writers call `llist_add()` without taking a lock at all; readers instead can use a spinlock or a mutex. 

`llist_del_first()` provides LIFO semantics for the `llist`. If your application requires FIFO order, however, there is a useful trick that you can apply, and that's where [`llist_reverse_order()`](<https://elixir.bootlin.com/linux/v5.11.2/source/lib/llist.c#L72>) comes into play. Removing a batch of items with `xchg()` (as is done with [`llist_del_all()`](<https://elixir.bootlin.com/linux/v5.11.2/source/include/linux/llist.h#L227>)) does provide the batches in FIFO order, only the items in each batch are ordered back to front. The following algorithm then comes to mind: 

```
struct thing *first, *current_batch;
    
        if (current_batch == NULL) {
            current_batch = xchg_acquire(&first, NULL);
            _... reverse the order of the nodes in current_batch ..._
        }
        node = current_batch;
        current_batch = current_batch->next;
```

Every execution of the previous pseudocode will return an element of the linked list in FIFO order. This is also a single-consumer data structure, as it assumes that only a single thread accesses `current_batch` at any given time. It is left as an exercise for the reader to convert the pseudocode to the `llist` API. 

That is all for this installment. The next article in this series will continue exploring read-modify-write operations, how to build them from compare-and-swap, and how they can be put into use to speed up reference-counting operations.
