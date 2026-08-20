---
title: "Time to move to C11 atomics?"
url: https://lwn.net/Articles/691128/
date: "June 15, 2016"
category: C11 atomic operations
author: "By Jonathan Corbet June 15, 2016"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 15, 2016

A typical program written in C looks like a deterministic set of steps laid out in a specific order. Out of the programmer's sight, though, both the compiler and the CPU are free to change the ordering of operations with the goal of speeding the program's execution. When one is dealing with a single thread of execution, reordering operations without breaking the program is a relatively straightforward task; that is no longer true when multiple threads are working with the same memory. In the multi-threaded case, developers must often make the ordering requirements explicit. 

To that end, the kernel has defined a whole set of memory barriers and atomic operations designed to preserve memory-access ordering in places where it matters while preserving performance. The C11 version of the C language tries to solve the same problems with a different set of barrier operations. Once again, the question has been asked: should the kernel drop its own operations in favor of those defined by the C standard? 

This question last came up in 2014; see [LWN's coverage of that discussion](<https://lwn.net/Articles/586838/>) for a great deal of background on how C11 atomic operations work and how concurrent memory access can go wrong when reordering of operations is not sufficiently controlled. This time around, compiler support for C11 atomic operations has improved, and David Howells has come forward with [a full implementation](<https://lwn.net/Articles/687863/>) of the (x86) kernel's atomic operations built on C11 atomics. The implementation itself is fairly straightforward; for example, the `atomic_read()` functions look like this: 

```
static __always_inline int __atomic_read(const atomic_t *v, int memorder)
        {
    	return __atomic_load_n(&v->counter, memorder);
        }
        #define atomic_read(v)		(__atomic_read((v), __ATOMIC_RELAXED))
        #define atomic_read_acquire(v)	(__atomic_read((v), __ATOMIC_ACQUIRE))
```

David's patches show that this conversion can be done; the real question is whether it _should_ be done. As one might expect, there are a number of arguments each way. 

Switching to C11 atomic operations would, in theory, allow the kernel to dump a bunch of tricky architecture-specific barrier code and take advantage of the same code, built into the compiler, that concurrent user-space programs will be using. C11 atomics give the compiler better visibility into what the code is actually doing, opening up more optimization possibilities and enabling the use of instructions that are tricky to invoke from inline assembly code. The compiler can also pick the instruction that is appropriate for the size of the operand; that can eliminate the big compile-time switch statements in the kernel's header files currently. 

The optimization possibilities are not fully realized with current compilers, but the potential exists for the compiler to, eventually, do better than even the most highly tweaked inline assembly code. As Paul McKenney [put it](<https://lwn.net/Articles/691289/>): 

I agree that might be very hard for the C11 intrinsics to beat tightly coded asms. But it might not be all that long before the compilers can beat straightforward hand-written assembly. And the compiler might well eventually be able to beat even tightly code asms in the more complex cases such as cmpxchg loops. 

There is also a benefit from the compiler being able to move specific barriers away from the actual atomic operation if that gives better performance; such moves are not possible with operations implemented in inline assembly. 

Of course, there are some disadvantages to making this switch as well. One of those is that C11 atomics are not implemented well in anything but the newest compilers. Indeed, David says that ""there will be some seriously suboptimal code production before gcc-7.1"" — a release that is not due for the better part of a year. As might be expected, numerous bugs involving C11 atomics have been turned up as part of this project; they are being duly reported and fixed, but there are probably more to come. In the long term, use of C11 atomics in the kernel would certainly lead to a more robust compiler implementation, but getting there might be painful. 

If a kernel built for multiprocessor operation (as almost all are) finds itself running on a uniprocessor system, it will patch the unneeded synchronization instructions out of its own code. If C11 atomics are used, this patching is not possible; it is no longer possible to know where those instructions are, and even small compiler changes could lead to massive confusion. Uniprocessor systems are increasingly rare and, arguably, custom kernels are already built for many of them, but it would still be better not to slow down such systems unnecessarily. 

Perhaps the biggest potential problem, though, is that the memory model implemented by C11 atomics does not exactly match the model used by the kernel. The C11 model is based on acquire/release semantics — one-way barriers that are described in the 2014 article and [this article](<https://lwn.net/Articles/576486/>). Much of the kernel, instead, makes use of load/store barriers, which are stricter, two-way barriers. A memory write with release semantics will only complete after any previous reads or writes are visible throughout the system, but it allows other operations made logically after the write to be reordered to happen before that write. A write with store semantics, instead, strictly orders other write operations on both sides of the barrier. 

One option would be to weaken the kernel's memory model so that architectures that have acquire/release semantics can gain the associated performance advantages. But, as one might imagine, such a change would be fraught with the potential for subtle, difficult-to-find bugs; it would have to be approached carefully. That said, David notes that the PowerPC seems to already be working with a weaker model, so there may not be many problems lurking in the core kernel. 

As Will Deacon [pointed out](<https://lwn.net/Articles/691295/>), C11 atomics lack a good implementation of [consume load operations](<https://lwn.net/Articles/588300/>), which are an important part of read-copy-update (RCU), among other things. A consume load can always be replaced with an acquire operation, but the performance will be worse. In general, Will worries that the C11 model is a poor fit for the ARM architecture, and that the result of a switch might be an unwieldy combination of C11 and kernel-specific operations. He did agree, though, that a generic implementation based on C11 atomics would be a useful tool for developers bringing up the kernel on a new architecture. 

There has, thus far, been far less discussion of this idea than happened last time around; perhaps developers are resigning themselves to the idea that this change will happen eventually, even if it seems premature now. There would certainly be advantages in such a switch, for both the kernel and the compiler communities. Whether those advantages justify the costs has not yet been worked out, though.
