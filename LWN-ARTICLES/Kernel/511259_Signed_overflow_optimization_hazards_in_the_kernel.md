---
title: Signed overflow optimization hazards in the kernel
url: https://lwn.net/Articles/511259/
date: "August 15, 2012"
category: GCC
author: "August 15, 2012 This article was contributed by Paul McKenney"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

August 15, 2012

This article was contributed by Paul McKenney

A recent [LWN article](<http://lwn.net/Articles/508991/>) described a couple of the hazards that compiler optimizations can pose for multithreaded code. This article takes a different approach, looking at a compiler-optimization hazard that can also strike sequential code. This hazard stems from an annoying aspect of the C11 standard, namely that signed-integer overflow is undefined (Section 3.4.3). 

Overflow is a consequence of the fact that a computer's native arithmetic capability is quite limited. For example, if a C program running on a typical Linux system tries adding one `int` variable with value 2,147,483,647 to another `int` with value 1, the result will be -2,147,483,648--which might surprise people who naively expect the mathematically correct value of +2,147,483,648. This deviation from mathematical correctness occurs because the machine cannot represent the correct value of 2,147,483,648 in a 32-bit [twos-complement](<http://en.wikipedia.org/wiki/Two's_complement>) integer. Therefore, any attempt to compute this number without [help](<http://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic>) from software will result in overflow. 

**Quick Quiz 1** : Yecch!!! Why can't CPU designers come up with something better?   
Answer

The number -2,147,483,648 is "unusual" in that adding it to itself (again, using twos-complement 32-bit integers) results in zero. Furthermore, it is its own negative: negating -2,147,483,648 results in -2,147,483,648. Therefore, this [weird number](<http://en.wikipedia.org/wiki/Two's_complement#The_most_negative_number>) is (1) equal to half of zero and (2) both positive and negative but (3) not equal to zero. This weirdness earned this number a special mention in the C standard, which says that its handling is implementation-defined (Section 6.2.6.2, Paragraph 2). 

Unfortunately, relying on signed integer overflow for both normal and weird values is extremely convenient when working with free-running counters. For example, suppose that our program is dealing with a succession of work items, each designated by an integer. Suppose further that this code might be called upon to report on the past and future 25 work items. This situation will likely require a fast way to distinguish between past, current, and future work. Signed twos-complement integers make this easy. If `current` is the integer corresponding to the current work item, 

```
(current - other > 0)
```

will evaluate to `true` if the `other` work item is from the past. This works, even if the counter hits the maximum value and wraps around, due to the circular nature of twos-complement integers: adding one to the largest positive number that can be represented results in the smallest negative number. As a result, there is no need to write special-case code to handle counter overflows. 

The simplicity of this approach has caused coders to willfully ignore the undefined nature of C signed-integer overflow since well before the dawn of Linux. For example, I was happily relying on twos-complement semantics from C signed-integer overflow in the early 1980s, and the only reason I wasn't doing so earlier was that I wasn't using C any earlier. Nor am I alone. Here are a couple of representative code fragments from version 3.5 of the Linux kernel: 

```
if (auth_vnode->acl_order - acl_order > 0) {
    
        return (int)(tcmp - __raw_readl(timer_base + MX1_2_TCN)) < 0 ?  -ETIME : 0;
```

The first is from `afs_cache_permit()` in the AFS filesystem, which is using this pattern to sort out the order of events in a distributed filesystem. The second example is from `mx1_2_set_next_event()` in the ARM architecture, which is using a variation on this theme to determine whether the requested event time really is in the future. Here the actual subtraction is unsigned, but the result is cast to a signed integer. Because unsigned longs are always positive, the only way that the result can be negative (when interpreted as a signed value) is overflow, which the compiler is permitted to assume never happens. The compiler is therefore within its rights to unconditionally evaluate the test as `false` and return zero, which might fatally disappoint the caller. 

In addition, there used to be several instances of this pattern in the Linux kernel's RCU implementation, where it was used to figure out whether a given request had been implicitly satisfied due to the efforts undertaken to fulfill concurrent requests of the same type. These have since been converted to use unsigned arithmetic using the technique described below. 

One might well ask: is there really a problem here? All systems running Linux are twos complement, so we really should not worry about clauses in the C standard designed to handle the wider variety of arithmetic that was available a few decades ago, right? 

Unfortunately, wrong. The C compiler can and does make use of undefined behavior when optimizing. To see this, consider the following code: 

```
1 long long_cmp_opt(const int a, const int b)
         2 {
         3   if (a > 0) {
         4     do_something();
         5     if (b < 0) {
         6       do_something_else();
         7       if ((a - b) > 0)
         8         do_another_thing();
         9     }
        10   }
        11 }
```

At line 7 the compiler knows that the variable `a` is positive and the variable `b` is negative. Therefore, ignoring the possibility of integer overflow, the compiler knows that this "if" condition will always evaluate to `true`, meaning that the compiler is within its rights to invoke `do_another_thing()` unconditionally, without actually doing the subtraction and comparison. In contrast, if `a` is (say) 2,147,483,647 and `b` is -2,147,483,648, the unoptimized code would avoid invoking `do_another_thing()`. Therefore, this optimization has significantly changed the program's behavior. 

**Quick Quiz 2** : But just how often is the compiler going to know the sign of both the values???   
Answer

Of course, in real life, overflow really can occur. But because the C standard says that signed overflow is undefined, the compiler is permitted to do whatever it wishes in the overflow case. And GCC 4.6.1 really does omit the subtraction and comparison when compiling this example for x86 at optimization levels of `-O2` or higher. 

Fortunately for the Linux kernel, GCC will generate the subtraction and comparison for `-O1` or less. But optimizations can migrate to lower optimization levels over time, and there may come a time when either performance or energy-efficiency considerations motivate the Linux kernel to move to higher optimization levels. If that happens, what can be done? 

**Quick Quiz 3** : First you were talking about overflowing, now about wrapping. Consistent terminology, please?   
Answer

One approach is to move to unsigned integers for free-running counters. The C standard defines unsigned integers to use modular arithmetic, so that wrapping the counter is fully defined (Section 6.2.5 Paragraph 9). 

Of course, checking for counter wrap must be done differently. For purposes of comparison, here is the (undefined) signed version: 

```
if ((a - b) < 0)
```

And here is the corresponding version for unsigned long types: 

```
if (ULONG_MAX / 2 < a - b)
```

This version relies on the fact that, bit for bit, twos-complement addition and subtraction are identical to their unsigned counterparts. Now, the bitwise representation of the constant `ULONG_MAX / 2` is a zero bit followed by all one-bits, which is the largest value that does not have the most-significant bit set. Therefore, if the result of computing `a - b` is greater than this constant, we know that this result has its uppermost bit set. Because the uppermost bit is the sign bit for twos-complement numbers, we are guaranteed that the signed and unsigned versions compute identical results. 

Of course, the unsigned version is more characters to type, but that is what inline functions and C-preprocessor macros are for. But what about the code that the compiler generates? After all, the Linux kernel absolutely does not need extra instructions loading large constants for each comparison! 

The good news is that GCC actually generates _exactly_ the same code for both of the above versions when compiled with `-O1` on both x86 and PowerPC: 

```
/* x86 code. */
          e:	8b 44 24 04          	mov    0x4(%esp),%eax
         12:	2b 44 24 08          	sub    0x8(%esp),%eax
         16:	c1 e8 1f             	shr    $0x1f,%eax
```

```
/* PowerPC code. */
         1c:   7c 64 18 50     subf    r3,r4,r3
         20:   78 63 0f e0     rldicl  r3,r3,1,63
```

Of course, there will be times when the Linux kernel absolutely must rely on undefined behavior. However, this is not one of those times: As shown above, there are straightforward ways to avoid relying on signed integer overflow. Removing the kernel's reliance on signed integer overflow could avoid our getting burned by increasingly aggressive optimization, and might further allow use of higher optimization levels to improve performance and battery lifetime. So it is not too early to start future-proofing the Linux kernel by removing its reliance on signed integer overflow! 

###  Answers to Quick Quizzes

**Quick Quiz 1** : Yecch!!! Why can't CPU designers come up with something better? 
    

**Answer** : CPU designers have come up with a [variety of schemes](<http://en.wikipedia.org/wiki/Signed_number_representations>) over the decades. However, in my experience, each scheme has its own peculiarities. I have used ones complement and twos complement, and dealing with the peculiarities of twos complement proved easier for me than those of ones complement. 

That said, I suspect that the dominance of twos complement was not due to ease of use, but rather due to the fact that it allows a single hardware adder to perform both signed and unsigned computations. 

**Back to Quick Quiz 1**.
**Quick Quiz 2** : But just how often is the compiler going to know the sign of both the values??? 
    

**Answer** : The more inline functions we add, the higher the probability that the compiler will be able to infer all sorts of things about the values in question, including their sign. And it only takes one unwanted optimization for the Linux kernel to fail. 

**Back to Quick Quiz 2**.
**Quick Quiz 3** : First you were talking about overflowing, now about wrapping. Consistent terminology, please? 
    

**Answer** : Interestingly enough, the C standard does not define overflow for unsigned integers. Instead, it defines the unsigned integral types to use modular arithmetic so as to eliminate the possibility of overflow. Aside from things like division by zero, that is. The term "wrap" works regardless. 

**Back to Quick Quiz 3**.
