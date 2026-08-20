---
title: Atomic primitives in the kernel
url: https://lwn.net/Articles/695257/
date: "July 27, 2016"
category: atomic t
author: "July 27, 2016 This article was contributed by Neil Brown"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

July 27, 2016

This article was contributed by Neil Brown

Since the Linux kernel typically has multiple threads of control running in parallel, quite possibly working on the same data, it is essential to have primitives that allow coordination among those threads. Additionally, it is beneficial to have a wide range of interfaces so that the right tool for any specific job is both easy to find and easy to use. It would not be fair to say that Linux has had a paucity of such primitives, but it has recently gained a generous helping of new operations anyway; that provides a convenient opportunity to have a look at what is available and why something new was needed.

**Composite primitives**  
Probably the most extreme example of a limited architecture is 32-bit SPARC processors, which only provide [an RMW operation that writes back all '1' bits after reading a value](<http://comp.mq.edu.au/%7Emike/comp226/sparc-manual/Load/ldstub.html>). This is sufficient for implementing a spinlock, but little else. To take a lock, the code can repeatedly use that instruction until it fetches a number that isn't all ones. This indicates that the current thread has gained the lock. It can then drop the lock by writing anything else to the address.

For SPARC-32, Linux implements all other synchronization primitives by first taking a spinlock, then performing the operation, then dropping the lock. It maintains a small hash table of locks, currently with four entries, and uses the address of the value being operated on to choose a lock to protect the operation. This allows some concurrency between non-conflicting synchronization operations and provides reasonable results on what is, today, an ancient architecture.

The class of synchronization primitives we are particularly interested in here is the set of read-modify-write (RMW) operations where the value in some memory location is read, modified in some way, and then written back, with the guarantee that no other write will occur to that location between the read and the write. Referring to these as "primitive" operations deserves a little explanation as primitiveness is a relative term. While some processors provide a variety of RMW operations using single instructions, others are more limited and require that each RMW operation be built out of component parts. 

Naturally any RMW operation could be performed by taking a lock, operating, then dropping the lock, but this is far from ideal, partly due to the need to have those locks at all, and partly because each operation then becomes at least three memory writes: lock, operate, and unlock. Fortunately, there is usually an alternative in the form of another construct that is nearly as general as a spinlock while not being quite so expensive: compare-exchange, or `cmp_xchg()`.

The compare-exchange operation takes an address and two values. It reads the contents of memory at the given address and, if the value found matches the first value, then the second value is stored, all as part of the one atomic read-modify-write operation. Whether the write happens or not, the value that was read is returned. This operation can be used to perform arbitrarily complex atomic updates while usually only requiring a single memory write. For example, an atomic increment, which reads the value, increments it, and writes out the new value, could be implemented with `cmp_xchg()` as:

```
do {
            int old = *addr;
            int new = old + 1;
        } while (cmp_xchg(addr, old, new) != old);
```

Compare-exchange is generally quite efficient and it is not hard to find a wide variety of compare-exchange loops like the above throughout the kernel. Building all atomic RMW operations out of compare-exchange would be possible, but would often not be ideal. For more modern and complex processors, many such operations can be performed with a single instruction, as mentioned already, and in those cases the single instruction would be the better choice. For older processors like the SPARC-32, this sort of loop would need to take and release a spinlock for each `cmp_xchg()` call, which is fairly pointless when the whole operation could be directly protected by the lock. 

For these reasons, Linux has a suite of different RMW operations that are implemented individually for each architecture. These are "primitives" from the perspective of a kernel developer writing architecture-independent code like a filesystem or a device driver. Some of them use `cmp_xchg()` loops, some use spinlocks, some use dedicated instructions, but most of us don't care.

#### Existing interfaces

While there are exceptions, most read-modify-write operations in Linux fall into one of two classes: those that operate on either the `atomic_t` or `atomic64_t` data type, and those that operate on bitmaps, either stored in an `unsigned long` or in an array of `unsigned long`, possibly declared with [the `DECLARE_BITMAP()` macro](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/types.h?id=92d21ac74a9e3c09b0b01c764e530657e4c85c49#n8>).

`atomic_t` and `atomic64_t` are effectively 32-bit and 64-bit numbers, respectively, that can only be operated on using the various `atomic_*()` interfaces, thus reducing the chances of careless mistakes. Apart from the trivial operations of setting an initial value and reading the current value, the most commonly used operations on these objects are to increment or decrement the value. Next most popular, by a fairly large margin, are `atomic_dec_and_test()` and `atomic_inc_return()`. These perform the same RMW operations, but also report the result of the operation, in one case comparing the result against zero first since that is the most common use case.

The most common RMW operations on a bitmap are `set_bit()` and `clear_bit()`, which set or clear one bit within the bitmap; to do so, they must read one word of the bitmap, modify that word, and write it back out again. The most common bitmap operation that returns a value is `test_and_set_bit()`, though `test_and_clear_bit()` is a close second. Again, these perform that same RMW operation as `set_bit()` and `clear_bit()` while also returning a value, but there is an important difference between the atomic and bitmap functions in the choice of the value returned.

For the `atomic_t` operations, the returned value is based on the value that was written to the memory address after the modification. For the bitmap operations, the returned value is based on the value that was read out of the memory address before the modification happened. For reversible operations like increment and decrement, this difference doesn't really matter as the original value can be found by reversing the operation, so "`atomic_inc_return(&v) - 1`" will report the original value in `v`. For bitmaps, instead, a `set_bit_return()` would be pointless as it would always report that the bit is now set.

Having inconsistent standards isn't ideal, but if all bitmap operations reported on the original value and all `atomic_t` operations reported on the result, it would be an easy enough convention to work with. However there is a growing desire to perform bitwise operations on `atomic_t` variables, resulting in [the addition](<https://lwn.net/Articles/651629/>) of `atomic_and()`, `atomic_or()`, and `atomic_xor()` in 2015 because a number of architectures were defining their own private versions. These three functions don't return a value, but what if one was wanted? Defining an `atomic_and_return()` that reported on the result wouldn't be particularly useful for much the same reason that `set_bit_return()` isn't, and having it report on the original value would be a confusing difference from `atomic_add_return()`.

This came to a head recently when a patch (by Frederic Weisbecker) to implement `fetch_or()` [was submitted by Ingo Molnar](<https://lwn.net/Articles/695735/>). `fetch_or(addr, mask)` would perform a bitwise "or" with the mask and the value fetched from the address, and the result would be written back in a single RMW operation. The value this function returns is the value fetched. This patch was sufficiently imperfect that it triggered a [response from Linus Torvalds](<https://lwn.net/Articles/695736/>) (""This is garbage."" — no-one disagreed) and a subsequent discussion of various issues surrounding the implementation, including the fact that it looked like it should be an `atomic_` operation but didn't operate on an `atomic_t`. I do sometimes wonder if posting imperfect patches has merit because of the increased discussion they can cause.

An important outcome of this discussion was a desire for a more general set of atomic operations that returned the fetched number instead of the stored number. As Peter Zijlstra had [a long airplane flight](<https://lwn.net/Articles/695737/>) shortly afterward, this desire soon became [a reality](<https://lwn.net/Articles/684775/>).

#### atomic_fetch_$OP()

The new set of operations, which has been implemented for every architecture supported by the kernel, each in the most appropriate style, has names constructed with the pattern: 

```
atomic{,64}_fetch_{add,sub,and,andnot,or,xor}()
```

In each case there are two arguments, the first being a number (`int` or `long`) and the second being a pointer to an atomic value (`atomic_t` or `atomic64_t` respectively). Thus, for example: 

```
atomic_fetch_add(v, addr);
```

will fetch the `atomic_t` value at `*addr`, add `v` to it, and store the result, returning the pre-addition value of `*addr`. 

This set of names intersects with the standard library functions that C11 defined for operating on the newly introduced atomic data types, so for example [the standard library also has `atomic_fetch_add()`](<http://en.cppreference.com/w/c/atomic/atomic_fetch_add>). We [previously looked at](<https://lwn.net/Articles/586838/>) the question of whether the C11 atomics might be useful in the Linux kernel and the conclusion at that time was that they probably wouldn't get used. This choice of conflicting names seems to have shifted us over into the "definitely will not be used" camp.

As a constant reminder that the Linux `atomic_fetch_add()` is not the C11 `atomic_fetch_add()`, the order of arguments has been reversed: C11 has the address of the atomic value first and the value used to modify it second. When Zijlstra was adding the new operations he [acknowledged that the order seemed backward to him](<https://lwn.net/Articles/695738/>) but, as we have an established pattern with `atomic_add()` and `atomic_sub()` having the atomic pointer second, it would be inappropriate to use a different order for the `fetch_` versions. ""I forever write: atomic_add(&v, val); and then have the compiler yell at me"" he reported, and it seems that it is just something we will all have to get used to.

#### The future of the `*_return()` interfaces

When [Zijlstra first hinted at his desire](<https://lwn.net/Articles/695739/>) for the new interfaces he described a vision that went even further:

I've even thought about reworking our entire atomic*_t bits to match. That is, introduce all the fetch_$op primitives, then convert all the $op_return ones over and finally remove all the $op_return ones. 

There has, as yet, been no movement from Zijlstra beyond the first step, presumably because he felt that ""people (and this would very much include me) would curse me for changing this"". Others haven't been quite so cautious. Last month Davidlohr Bueso [posted a patch series](<https://lwn.net/Articles/695740/>) that introduced `atomic_fetch_inc()` and `atomic_fetch_dec()` based on `atomic_fetch_add()` and `atomic_fetch_sub()`, and then proceeded to convert several call sites that currently use "`atomic_inc_return(&var) - 1`" over to using "`atomic_fetch_inc(&var)`".

As Zijlstra predicted, not everyone was happy about this change. James Bottomley, maintainer of some of the code that was being changed, [wanted to see some justification](<https://lwn.net/Articles/695741/>), and declared that if it was just to help the compiler when it cannot optimize away a subtraction, then he's ""not sure the added complexity justifies the cycle savings"".

The "added complexity" here is not complexity in the code, which is strictly simpler, but the complexity for those reading the code who would now have a new interface to learn and get used to. These atomic operations _are_ primitives in the sense that they are a core part of the particular dialect of C that we use to write the kernel. They are not something we want to have to think about every time we see them. 

Any language that is in regular use must be considered a living thing that will grow and change whether we want it to or not. Introducing new words with clear practical value makes a lot of sense. Imposing the use of those words on others who are quite comfortable with their current vocabulary doesn't make nearly as much sense. I have no doubt that we will see the use of these new interfaces spread through the kernel in places were code is being renewed, until eventually there may well be a strong case for deprecating the old functions once they are hardly used any more.
