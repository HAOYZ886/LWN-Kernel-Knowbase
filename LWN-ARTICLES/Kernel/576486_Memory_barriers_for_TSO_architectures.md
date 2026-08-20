---
title: Memory barriers for TSO architectures
url: https://lwn.net/Articles/576486/
date: "December 11, 2013"
category: Memory barriers
author: "By Jonathan Corbet December 11, 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 11, 2013

Developers — even kernel developers — like to think of access to memory as a nice, predictable, deterministic thing. If two memory locations are written in a given order, one would hope that any other processors would see those writes in the same order. The truth of the matter is rather murkier; compilers and processors are both happy to take liberties with the ordering of memory accesses in the name of performance. Most of this playing around is invisible to programmers, but it can interfere with the correct operation of concurrent systems, so developers must occasionally force things to happen in the right order through the use of memory barriers. The set of available barrier types looks like it will get a bit larger in the 3.14 kernel. 

#### Introduction to memory barriers

A memory barrier is a directive that prohibits the hardware (and compiler) from reordering operations in specific ways. To see how they might be used, consider the following simple example, taken from [a 2013 Kernel Summit session](<https://lwn.net/Articles/572099/>). The lockless insertion of a new element into a linked list can be performed in two steps. The first is to set the "next" pointer of the new item to point to the item that will follow it in the list: 

> ![\[Linked list\]](https://static.lwn.net/images/conf/2013/lce-ks/rcu-ll2.png)

Once that is done, the list itself can be modified to include the new item: 

> ![\[Linked list\]](https://static.lwn.net/images/conf/2013/lce-ks/rcu-ll3.png)

A thread walking the list will either see the new item or it won't, depending on the timing, but it will see a well-formed list in either case. If, however, the operations are reordered such that the second pointer assignment becomes visible before the first, there will be a period of time during which the structure of the list is corrupted. Should a thread follow that pointer at the wrong time, it will end up off in the weeds. To keep that from happening, this sort of list operation must use a memory barrier between the two writes. With a proper barrier in place, the pointer assignments will never be seen in the wrong order. 

The kernel offers a wide variety of memory barrier operations adapted to specific situations, but the most commonly used barriers are: 

  * `smp_wmb()` is a _write_ memory barrier; it ensures that any write operation executed after the barrier will not become visible until all writes executed prior to the barrier are visible. A write memory barrier would be the appropriate type to use in the linked list example above. 

  * `smp_rmb()` is a _read_ memory barrier; any reads executed before the barrier are forced to complete before any reads after the barrier can happen. Code traversing a linked list that is subject to lockless modification would want to use read barriers between access to subsequent link pointers. 

  * `smp_mb()` is a barrier for both read and write operations; it can be thought of as the combination of `smb_wmb()` and `smp_rmb()`. 

Memory barriers almost invariably come in pairs. If one of two cooperating threads cares about the order in which two values are written, the other side must be equally concerned about the order in which those values are read. 

Naturally enough, the full story is rather more complex than described here. Readers with sufficient interest and free time, along with quite a bit of excess brain power, can read [Documentation/memory-barriers.txt](<https://lwn.net/Articles/576489/>) for the full story. 

The primary reason for the proliferation of memory barrier types is performance. A full memory barrier can be an expensive operation; that is something that kernel developers would prefer to avoid in fast paths. Weaker barriers are often cheaper, especially if they can be omitted altogether on some architectures. The x86 architecture, in particular, offers more ordering guarantees than some others do, making it possible to do without barriers entirely in some situations. 

#### TSO barriers

A situation that has come up relatively recently has to do with "total store order" (TSO) architectures, where, as Paul McKenney [put it](<https://lwn.net/Articles/576492/>), ""reads are ordered before reads, writes before writes, and reads before writes, but not writes before reads"". The x86 architecture has this property, though some others do not. TSO ordering guarantees are enough for a number of situations, but, in current kernels, a full memory barrier must be used to ensure those semantics on non-TSO architectures. Thus, it would be nice to have yet another memory barrier primitive to suit this situation. 

Peter Zijlstra had originally called the new barrier `smp_tmb()`, but Linus was [less than impressed](<https://lwn.net/Articles/576493/>) with the descriptive power of that name. So Peter came up with [a new patch set](<https://lwn.net/Articles/576494/>) adding two new primitives: 

  * `smp_load_acquire()` forces a read of a location in memory (in much the same way as [`ACCESS_ONCE()`](<https://lwn.net/Articles/508991/>)), but it ensures that the read happens before any subsequent reads or writes. 

  * `smp_store_release()` writes a value back to memory, ensuring that the write happens after any previously-executed reads or writes. 

These new primitives are immediately put to work in the code implementing the ring buffer used for perf events. That buffer has two pointers, called `head` and `tail`; `head` is where the kernel will next write event data, while `tail` is the next location user space will read events from. Only the kernel changes `head`, while only user space can change `tail`. In other words, it is a fairly standard circular buffer. 

The code on the kernel side works like this (in pseudocode form): 

```
tail = smp_load_acquire(ring_buffer->tail);
        write_events(ring_buffer->head); /* If 'tail' indicates there is space */
        smp_store_release(ring_buffer->head, new_head);
```

The `smp_load_acquire()` operation ensures that the proper tail pointer is read _before_ any data is written to the buffer. And, importantly, `smp_store_release()` ensures that any data written to the buffer is actually visible there before the new `head` pointer is made visible. Without that guarantee, the reader side could possibly see a `head` pointer indicating that more data is available before that data is actually visible in the buffer. 

The code on the read side is the mirror image: 

```
head = smp_load_acquire(ring_buffer->head);
        read_events(tail);  /* If 'head' indicates available events */
        smp_store_release(ring_buffer->tail, new_tail);
```

Here, the code ensures that the `head` pointer has been read before trying to access any data in the buffer; in that way, `head` corresponds to the data the kernel side wrote there. This `smp_load_acquire()` operation is thus paired with the `smp_store_release()` in the kernel-side code; together they make sure that data is seen in the correct order. The `smp_store_release()` call here pairs with the `smp_load_acquire()` call in the kernel-side code; it makes sure that the `tail` pointer does not visibly change until user space has fully read the data from the buffer. Without that guarantee, the kernel could possibly overwrite that data before it was actually read. 

The ring buffer code worked properly before the introduction of these new operations, but it had to use full barriers, making it slower than it needed to be. The new operations allow this code to be optimized while also better describing the exact operations that are being protected by barriers. As it happens, a lot of kernel code may be able to work with the slightly weaker guarantees offered by the new barrier operations; the patch changelog says ""It appears that roughly half of the explicit barriers in core kernel code might be so replaced."" 

The cost, of course, is that the kernel's complicated set of memory barrier operations has become even more complex. Once upon a time that might not have mattered much, since most use of memory barriers was deeply hidden within other synchronization primitives (spinlocks and mutexes, for example). With scalability pressures pushing lockless techniques into more places in the kernel, though, the need to be explicitly aware of memory barriers is growing. There may come a point where understanding memory-barriers.txt will be mandatory for working in much of the kernel.
