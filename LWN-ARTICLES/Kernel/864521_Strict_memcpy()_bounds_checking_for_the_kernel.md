---
title: Strict memcpy() bounds checking for the kernel
url: https://lwn.net/Articles/864521/
date: "July 30, 2021"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet July 30, 2021"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 30, 2021

The C programming language is famously prone to memory-safety problems that lead to buffer overflows and a seemingly endless stream of security vulnerabilities. But, even in C, it is possible to improve the situation in many cases. One of those is the `memcpy()` family of functions, which are used to efficiently copy or overwrite blocks of memory; with a bit of help from the compiler, those functions can be prevented from writing past the end of the destination object they are passed. Enforcing that condition in the kernel is harder than one might expect, though, as [this massive patch set](<https://lwn.net/ml/linux-kernel/20210727205855.411487-1-keescook@chromium.org/>) from Kees Cook shows. 

Buffer overflows never seem to go away, and they are a constant source of bugs and security problems in the kernel. That said, hardening techniques have become good enough that many types of stack-based overflows can be detected and defended against (by killing the system if nothing else). It is hard to overwrite the stack without running over boundaries (which may contain a canary value) in ways that make the problem evident. Heap-based data lacks such boundaries, though, making overflows in the heap space harder to detect; as a result, attackers tend to find such vulnerabilities attractive. 

#### Fortifying the source

The kernel's `FORTIFY_SOURCE` configuration option turns on a range of checks for functions that are commonly implicated in memory problems in the heap area (and beyond). The `strcpy()` family of functions, for example, is fairly thoroughly checked when this option is turned on. There are also checks for `memcpy()` and friends; consider the fortified version of `memset()` from [include/linux/fortify-string.h](<https://elixir.bootlin.com/linux/v5.13.5/source/include/linux/fortify-string.h>) which, in current kernels, looks like this: 

```
__FORTIFY_INLINE void *memset(void *p, int c, __kernel_size_t size)
        {
    	size_t p_size = __builtin_object_size(p, 0);
    
    	if (__builtin_constant_p(size) && p_size < size)
    	    __write_overflow();
    	if (p_size < size)
    	    fortify_panic(__func__);
    	return __underlying_memset(p, c, size);
        }
```

This version asks the compiler for the size of the destination object (`p`). If the passed-in `size` is known at compile time (the `__builtin_constant_p()` test is true), then the test can be made right away, causing compilation to fail if an overflow is detected; otherwise the second `if` test performs the check at run time. Note that the run-time test will be optimized out by the compiler in cases where the size is known to be within bounds. 

So it would seem that the kernel already has bounds checking for these functions, but there's a catch. The second argument to `__builtin_object_size()` describes _which_ object is of interest. This comes into play when, for example, the object of interest is embedded within a structure. If that second argument is zero (as in the example above), the return size is the number of bytes to the end of the containing structure; setting that argument to one, instead, gives only the size of the immediate object itself. See [the GCC documentation](<https://gcc.gnu.org/onlinedocs/gcc/Object-Size-Checking.html>) for more information on `__builtin_object_size()`. 

The end result is that the version of `memset()` shown above will catch attempts to write beyond the end of a structure, but will not catch overflows that overwrite structure fields after the intended destination. That leaves a lot of interesting fields for an attacker to step on if they can find a way to influence the size passed into those functions. One might think that the obvious thing to do is to change the second argument to `__builtin_object_size()` to one, thus checking against the correct size, but this is the kernel and life is not so simple. 

Setting or copying data across multiple structure fields is, as it turns out, a fairly common action in the kernel, and those actions would trigger more strict tests in the memory functions. The result of enabling the strict tests would be an unbuildable, unusable kernel; that would certainly be secure, but users would still be unimpressed. Users can be a little funny that way. 

#### `memset_after()`

One common use case for copying across fields is the "write zeroes from here to the end of the structure" operation. Consider, for example, [this code](<https://elixir.bootlin.com/linux/v5.13.5/source/drivers/net/wireless/ath/carl9170/tx.c#L280>) in the AR9170 wireless network driver: 

```
memset(&txinfo->status.ack_signal, 0,
    	   sizeof(struct ieee80211_tx_info) -
    	   offsetof(struct ieee80211_tx_info, status.ack_signal));
```

This code shows a case of clearing to the end of the structure; it also shows just how awkward such code can be. That sort of length arithmetic is easy to get wrong, and it's subject to disruption if the layout of the structure changes for any reason. Indeed, the line of code _before_ the above reads: 

```
BUILD_BUG_ON(offsetof(struct ieee80211_tx_info, status.ack_signal) != 20);
```

This test will cause a build failure if the offset of the first field to overwrite is not as expected, but will not catch any changes made after that field. Structure members added after `ack_signal` will be overwritten by this `memset()` call — a fact that may not be obvious at the time. 

To clarify this sort of code and to avoid false positives from stricter checks on `memset()`, the patch set [introduces a new macro](<https://lwn.net/ml/linux-kernel/20210727205855.411487-38-keescook@chromium.org/>) for this operation: 

```
memset_after(object, value, member);
```

It will cause every byte of `object` located after `member` to be set to `value`. This macro can replace the above code with: 

```
memset_after(&txinfo->status, 0, rates);
```

(The `ack_signal` field, being the first to be zeroed, is immediately after `rates` in this structure). Numerous such cases have been fixed in Cook's patch set. 

#### Grouped structure fields

There is a more complicated case, though, in which a range of fields within a structure is overwritten in a single call. A number of approaches have been used within the kernel to try to do such copies safely; one of those is the same sort of `offsetof()` arithmetic seen in the case above. But there are others. [Deep within the `sk_buff` structure](<https://elixir.bootlin.com/linux/v5.13.5/source/include/linux/skbuff.h#L798>) used to represent network packets is this field: 

```
__u32 headers_start[0];
```

[A full 120 lines later](<https://elixir.bootlin.com/linux/v5.13.5/source/include/linux/skbuff.h#L918>) is another zero-length array called `headers_end`. Those arrays clearly cannot hold any data of interest; instead, they are used with the same sort of offset arithmetic to copy a whole set of packet headers in a single operation. Here, too, there is [a set of build-time checks](<https://elixir.bootlin.com/linux/v5.13.5/source/net/core/skbuff.c#L974>) to ensure that, at least, all of the relevant header fields are located between the two markers. 

Some developers simply add up the lengths of the fields to be written and use the result as the length for the memory operation. Yet another approach is to define a nested structure to hold the set of fields to be copied. This variant is safer, but it complicates the use of those fields (which must be accessed by way of the intermediate structure) and tends to lead to pollution of the namespace with macros added to minimize those complications. 

In summary, kernel developers have come up with a number of ways of handling cross-field memory operations, but none of them are particularly satisfying. Cook's patch set brings a new solution (co-authored with Keith Packard) in the form of [the `struct_group()` macro](<https://lwn.net/ml/linux-kernel/20210727205855.411487-5-keescook@chromium.org/>). Taking the example from that patch, consider a structure like this: 

```
struct foo {
    	int one;
    	int two;
    	int three;
    	int four;
        };
```

Imagine further that the developer wants to copy over fields `two` and `three` with a single `memcpy()` call. This could be formalized by declaring the structure this way: 

```
struct foo {
    	int one;
    	struct_group(thing,
    		     int two,
    		     int three,
    	);
    	int four;
        };
```

This macro has the effect of creating a nested structure called `thing`, which can be used with functions like `memcpy()` with the strict bounds checks enabled. The individual fields can still be referred to as `two` and `three`, though, without the need to name the nested structure, and without any macro ugliness. This is accomplished this way: 

```
#define struct_group_attr(NAME, ATTRS, MEMBERS) \
    	union { \
    	    struct { MEMBERS } ATTRS; \
    	    struct { MEMBERS } ATTRS NAME; \
    	}
    
        #define struct_group(NAME, MEMBERS)	\
    	struct_group_attr(NAME, /* no attrs */, MEMBERS)
```

This macro defines an intermediate structure to hold the grouped fields — twice; one is anonymous while the other has the given `NAME`. The duplicated structures are then overlaid on top of each other within an anonymous union. This bit of trickery makes it possible to use the field names directly while also providing the name for the structure as a whole, which can be used with the memory functions. 

#### Toward a harder kernel

Much of the patch set is devoted to defining these groups within structures throughout the kernel, then using the groups for memory operations. With that done, it becomes possible to enable the stricter bounds checks for those operations — sort of. The remaining problem is that this kind of cross-field operation is actually kind of hard to find in the code; there is not a pattern that can be easily grepped for. Chances are thus good that there are other occurrences in the kernel that have not been found yet; as Cook [noted](<https://lwn.net/ml/linux-kernel/20210727205855.411487-35-keescook@chromium.org/>) halfway through the patch series, there are over 25,000 `memcpy()` calls in the kernel. Crashing the system in response to an unfixed (but correct) cross-field operation would be seen as rude at best, so warnings will have to be issued instead for the indefinite future. 

There should come a time, though, when reports of warnings fall off and the community will feel confident enough to halt the system when an out-of-bounds copy is detected. The value of doing so could be significant. Quoting the just-linked patch: 

> With this it's also possible to compare the places where the known 11 memcpy() flaw overflows happened against the resulting list of potential new bounds checks, as a measure of potential efficacy of the tightened mitigation. Much to my surprise, horror, and delight, all 11 flaws would have been detected by the newly added run-time bounds checks, making this a distinctly clear mitigation improvement. 

This mitigation seems worth having, but first the patches must find their way into the mainline kernel. Security-related work often has a rough path into the kernel, though the situation has gotten better over the years. In this case, at least, one frequent complaint (impact on performance) should not be an issue; the cost of an extra length check in the cases where the answer isn't known at compile time is tiny. But the patch set is still large and wide-ranging; chances are that there will be some discussions to get through before it can be merged. The completion of that process should herald the end of another type of unpleasant security bugs.
