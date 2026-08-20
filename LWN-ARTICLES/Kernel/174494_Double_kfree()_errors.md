---
title: Double kfree() errors
url: https://lwn.net/Articles/174494/
date: "March 6, 2006"
category: Debugging
author: 
---

Less than 24 hours after Coverity announced the availability of a new set of machine-detected potential kernel bugs, Dave Jones started posting fixes. Judging from these fixes, a number of the problems detected this time around are double-free errors - passing the same pointer to `kfree()` twice. Freeing memory twice is a sure way to corrupt core kernel data structures, leading to trouble in unpredictable places far from where the real bug is to be found. Avoiding this kind of error would make life easier for everybody involved. 

To that end, Dave [tossed out](<https://lwn.net/Articles/174496/>) a simple idea: have `kfree()` poison pointers so that a second call can be detected immediately. His first proposal looked like this: 

```
#define kfree(foo) \
    	        __kfree(foo); \
    	        foo = KFREE_POISON;
```

This code was not meant to be incorporated as-is; for starters, it probably needs a pair of braces. But there were a couple of other problems which popped up. One of them is that, since passing a NULL pointer to `kfree()` is legal, passing it twice is also legal. But this code would break that case. Whether that would be a problem for real code is unclear. Al Viro [pointed out](<https://lwn.net/Articles/174498/>) a more serious issue: the pointer passed to `kfree()` is not always an lvalue which can be assigned to. So simply redefining `kfree()` in this way would lead to compilation errors. 

The end result is that a transparent, in-place replacement for `kfree()` may be hard to implement. An alternative might be the creation of a `safe_kfree()` variant, combined with some serious pressure to use that variant. Then, perhaps, double-free errors could be caught when they happen. 

Or, instead, one could use the double-free checking already built into the kernel. The slab allocator, which is (among other things) the engine behind `kmalloc()` and `kfree()`, has options for poisoning (writing special values to) all memory which it handles. One value (`0x5a` in every byte) marks uninitialized memory, while another (`0x6b`) is written into memory when it is freed. The resulting patterns jump out nicely in oops listings, often making the cause of the problem immediately obvious. But the use-after-free value can also enable the detection of double-free errors - assuming that the memory is not reallocated between `kfree()` calls. 

The problem, it seems, is that not a whole lot of developers are running with slab poisoning enabled. As a result, they are working without a valuable debugging tool and allowing certain kinds of bugs to persist in the code base. So a part of the solution to the problem may well be a stronger effort to get developers to turn the slab poisoning option on. Beyond that, any sort of checking added to `kfree()` (or a variant) should be harder to disable than the existing debugging options.
