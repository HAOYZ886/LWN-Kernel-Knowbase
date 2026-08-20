---
title: "Variable-length arrays and the max() mess"
url: https://lwn.net/Articles/749064/
date: "March 12, 2018"
category: "Variable-length arrays"
author: "By Jonathan Corbet March 12, 2018"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 12, 2018

Variable-length arrays (VLAs) have a non-constant size that is determined (and which can vary) at run time; they are supported by the ISO C99 standard. Use of VLAs in the kernel has long been discouraged but not prohibited, so there are naturally numerous VLA instances to be found. A recent push to remove VLAs from the kernel entirely has gained momentum, but it ran into an interesting snag on the way. 

A VLA is simply an array that is declared with a non-constant dimension. For example, [`btree_merge()`](<https://elixir.bootlin.com/linux/latest/source/lib/btree.c#L639>) begins like this: 

```
int btree_merge(struct btree_head *target, struct btree_head *victim,
    		    struct btree_geo *geo, gfp_t gfp)
        {
    	unsigned long key[geo->keylen];
    	unsigned long dup[geo->keylen];
```

The length of both the `key` and `dup` arrays is determined by the value stored in the `keylen` field of the passed-in `geo` structure. The compiler cannot know what that value will be at compile time, so those arrays must be allocated at run time. For this reason, VLAs must be automatic variables, allocated on the stack. 

As an extension to the language, GCC also allows the use of VLAs within structures. The LLVM Clang developers have refused to add this extension, though; that has led developers interested in building the kernel with Clang to work to remove VLAs from structures in the kernel. C99 VLAs are supported by Clang, though, so developers working on Clang compatibility have not paid much attention to them. 

Since VLAs are standard C, one might wonder why there is a desire to remove them. One reason is that they add a bit of run-time overhead, since the size of a VLA must be calculated every time that the function declaring it is called. But the bigger issue has to do with stack usage. Stacks in the kernel are small, so every kernel developer must be constantly aware of how much stack space each function is using. VLAs, since they are automatic variables whose size is determined at run time, add a degree of uncertainty to a function's stack usage. Changes in distant code might result in a VLA growing in surprising ways. If an attacker finds a way to influence the size of a VLA, the potential for all kinds of mischief arises. 

For these reasons, Linus Torvalds recently [declared](<https://lwn.net/Articles/749089/>) that ""using VLA's is actively bad not just for security worries, but simply because VLA's are a really horribly bad idea in general in the kernel"". That added some energy to the VLA-removal work that was already underway. In the process, though, Kees Cook discovered an interesting surprise: a number of VLAs in the kernel are not actually variable in length and were never meant to be seen as such by the compiler. 

#### Accidental VLAs and the difficult story of max()

A useful tool in the quest to remove VLAs from the kernel is the GCC `-Wvla` option, which issues warnings when VLAs are declared. Cook [found](<https://lwn.net/Articles/749091/>), though, that it was issuing warnings for arrays that were meant to be of constant size; one of them was this bit of code from [`lib/vsprintf.c`](<https://elixir.bootlin.com/linux/latest/source/lib/vsprintf.c#L741>): 

```
#define RSRC_BUF_SIZE	((2 * sizeof(resource_size_t)) + 4)
        #define FLAG_BUF_SIZE	(2 * sizeof(res->flags))
        #define DECODED_BUF_SIZE	sizeof("[mem - 64bit pref window disabled]")
        #define RAW_BUF_SIZE	sizeof("[mem - flags 0x]")
    	char sym[max(2*RSRC_BUF_SIZE + DECODED_BUF_SIZE,
    		     2*RSRC_BUF_SIZE + FLAG_BUF_SIZE + RAW_BUF_SIZE)];
```

The length of `sym` is clearly constant and can be determined at compile time, but GCC warns that `sym` is a VLA anyway. The problem turns out to be the kernel's `max()` macro, which generates an expression that is not recognized as constant by the compiler. 

If one looks on page 87 of the original edition of [_The C  Programming Language_](<https://archive.org/details/TheCProgrammingLanguageFirstEdition>) by Kernighan and Ritchie, one will find the classic definition of the `max()` macro: 

```
#define max(A, B)  ((A) > (B) ? (A) : (B))
```

There are some problems with this definition that make it unsuitable for kernel use, including the double-evaluation of the arguments and the lack of any sort of type checking. So a lot of effort has gone into the kernel's [special version of `max()`](<https://elixir.bootlin.com/linux/v4.15.9/source/include/linux/kernel.h#L804>): 

```
#define __max(t1, t2, max1, max2, x, y) ({		\
    	t1 max1 = (x);					\
    	t2 max2 = (y);					\
    	(void) (&max1 == &max2);			\
    	max1 > max2 ? max1 : max2; })
    
        #define max(x, y)					\
    	__max(typeof(x), typeof(y),			\
    	      __UNIQUE_ID(max1_), __UNIQUE_ID(max2_),	\
    	      x, y)
```

For the curious, the outer `max()` macro uses `[__UNIQUE_ID()](<https://elixir.bootlin.com/linux/v4.15.9/source/include/linux/compiler-gcc.h#L189>)` to generate two unique variable names. Those names, along with the types of the two values and the values themselves, are passed to `__max()`. That macro declares two new variables (using the unique names) and assigns the passed-in values to them; this is done to prevent `x` and `y` from being evaluated more than once. Pointers to those two variables are then compared while discarding the result; this is done to force the compiler to issue an error if the two operands do not have compatible types. Finally, the two values themselves are compared to determine which one is greater. 

It was initially assumed that GCC was simply not smart enough to understand that the result of this whole mess was still constant, so it created a VLA instead. Torvalds eventually [figured out](<https://lwn.net/Articles/749093/>) the real problem, though: the C standard makes a distinction between a "constant value" and a "constant expression". Array dimensions are required to be constant expressions, but the `max()` macro does not qualify as such. As Torvalds pointed out, the warning from GCC is thus somewhat misleading; while the compiler is emitting the VLA code for these arrays, they are not actually variable in length, and the problem is more a matter of syntax. 

Regardless of the reason for their creation, it clearly would be a good thing to get rid of these "accidental" VLAs. What followed was an effort to rewrite `max()`; it was the sort of quest that the kernel community is so good at mustering. At one point, Cook [came up with this](<https://lwn.net/Articles/749095/>): 

```
#define __single_eval_max(t1, t2, max1, max2, x, y) ({	\
     	t1 max1 = (x);					\
     	t2 max2 = (y);					\
     	(void) (&max1 == &max2);			\
     	max1 > max2 ? max1 : max2; })
    
        #define __max(t1, t2, x, y)						\
    	__builtin_choose_expr(__builtin_constant_p(x) &&		\
    			      __builtin_constant_p(y),			\
    			      (t1)(x) > (t2)(y) ? (t1)(x) : (t2)(y),	\
    			      __single_eval_max(t1, t2,			\
    						__UNIQUE_ID(max1_),	\
    						__UNIQUE_ID(max2_),	\
    						x, y))
    
        #define max(x, y)	__max(typeof(x), typeof(y), x, y)
```

Essentially, this version uses more macro trickery to short out much of the `max()` logic when the two operands are constants. It worked well — on recent compilers. The results turned out to not be so pleasing on older compilers, though. Initially there was some thought of revisiting the discussion on [deprecating support for older compilers](<https://lwn.net/Articles/748074/>), but then 4.8.5 was [reported](<https://lwn.net/Articles/749097/>) to fail as well. At that point, Cook [threw up his hands](<https://lwn.net/Articles/749096/>) and gave up on creating a better `max()`. The solution going forward is likely to be what he suggested when he first discovered the problem: create a new `SIMPLE_MAX()` macro that is really just the original Kernighan and Ritchie `max()` with a different name. It is good enough for constant array dimensions. 

#### Finishing the job

Now that this discussion has run its course, the process of eliminating all of the non-accidental VLAs can continue. There are just over 200 of them in the kernel; many of them are easily replaced with ordinary fixed-length arrays. Several developers have posted patches eliminating VLAs from various kernel subsystems. Getting rid of all of them will probably take some time; a few instances may require fairly deep analysis to determine the real maximum length and, perhaps, internal API changes. Eventually, though, we will get to a point where a `-Wvla` build generates no warnings.
