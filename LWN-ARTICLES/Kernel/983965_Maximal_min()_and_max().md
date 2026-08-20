---
title: Maximal min() and max()
url: https://lwn.net/Articles/983965/
date: "August 1, 2024"
category: Build system
author: "By Jonathan Corbet August 1, 2024"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 1, 2024

Like many projects written in C, the kernel makes extensive use of the C preprocessor; indeed, the kernel's use is rather more extensive than most. The preprocessor famously has a number of sharp edges associated with it. One might not normally think of increased compilation time as one of them, though. It turns out that some changes to a couple of conceptually simple preprocessor macros — `min()` and `max()` — led to some truly pathological, but hidden, behavior where those macros were used. 

#### `min()` and `max()` for the kernel

Your editor's well-worn, first-edition copy of _The C Programming Language_ introduces the preprocessor with this example: 

```
#define max(A, B) ((A) > (B) ? (A) : (B))
```

The hazards that come with a macro like this, such as the double evaluation of the arguments, were pointed out in the text. Still, that did not prevent kernel developers from making use of it; as [covered here](</2001/0823/kernel.php3>) in 2001, there were over 150 definitions of `min()` and `max()` matching the above pattern in the 2.4.8 kernel. 

At that time, Linus Torvalds decided that a consolidation made sense; he added a single set of those macros meant to be used throughout the kernel. He also changed the interface, though, adding a type parameter describing how the comparison is to be performed — signed or unsigned integer, for example. The goal was to increase correctness, but the immediate effect was to break compilation throughout the kernel; the result was [a classic linux-kernel flame war](<https://lore.kernel.org/lkml/20010824224232.52238.qmail@web10908.mail.yahoo.com/>) of the type that, fortunately, tends not to happen anymore. 

Despite the complaints, the changes stuck — briefly. When the 2.4.9.8 release came about in February 2002, it included a change described as: ""make the three-argument (that everybody hates) 'min()' be 'min_t()', and introduce a type-anal 'min()' that complains about arguments of different types"". The `max()` and `min()` macros were back to their old form, but the definition had changed; they now looked like: 

```
#define min(x,y) ({ \
    	const typeof(x) _x = (x);	\
    	const typeof(y) _y = (y);	\
    	(void) (&_x == &_y);		\
    	_x < _y ? _x : _y; })
```

Unsurprisingly, the complexity of these macros only grew from there as developers added more features for flexibility and type safety. Numerous variants have also been added for special cases. Recently, [this series from David Laight](<https://lwn.net/ml/linux-kernel/b97faef60ad24922b530241c5d7c933c@AcuMS.aculab.com/>), merged for the 6.7 kernel, made `min()` and `max()` work properly in numerous cases where the two arguments have different types. All seemed well after that, and nobody felt a compelling urge to change these macros for at least three development cycles. 

#### Maximal expansion

But, then, Arnd Bergmann [observed](<https://social.kernel.org/notice/AkDuGHsn0WuA1g1uD2>) that the time required to compile the kernel had grown considerably in recent releases, and that the preprocessor had a lot to do with it; one file took a full 15 seconds just to get through the preprocessor stage. The problem came down to a single line of code in [`arch/x86/xen/setup.c`](<https://elixir.bootlin.com/linux/v6.10.2/source/arch/x86/xen/setup.c#L758>): 

```
extra_pages = min3(EXTRA_MEM_RATIO * min(max_pfn, PFN_DOWN(MAXMEM)),
        		       extra_pages, max_pages - max_pfn);
```

To see how this came about, it is worth looking at the 6.10 definitions of the `min()` and `max()` macros and their variants, all of which come from [`include/linux/minmax.h`](<https://elixir.bootlin.com/linux/v6.10.2/source/include/linux/minmax.h>). To start with, `min3()` returns the minimum of three values; its implementation is straightforward enough: 

```
#define min3(x, y, z) min((typeof(x))min(x, y), z)
```

That uses our old friend `min()`; indeed, it nests one `min()` call inside another. In 6.10, `min()` looks like this: 

```
#define min(x, y) __careful_cmp(min, x, y)
```

The `__careful_cmp()` macro tries hard to perform a type-safe comparison while evaluating the arguments only once; it also endeavors to expand to a constant expression if its arguments are constant expressions. That leads to a certain amount of complexity, implemented this way (best read from bottom to top): 

```
#define __cmp_op_min <
        #define __cmp_op_max >
    
        #define __cmp(op, x, y)	((x) __cmp_op_##op (y) ? (x) : (y))
    
        #define __cmp_once_unique(op, type, x, y, ux, uy) \
    	({ type ux = (x); type uy = (y); __cmp(op, ux, uy); })
    
        #define __cmp_once(op, type, x, y) \
    	__cmp_once_unique(op, type, x, y, __UNIQUE_ID(x_), __UNIQUE_ID(y_))
    
        #define __careful_cmp_once(op, x, y) ({			\
    	static_assert(__types_ok(x, y),			\
    		#op "(" #x ", " #y ") signedness error, fix types or consider u" #op "() before " #op "_t()"); \
    	__cmp_once(op, __auto_type, x, y); })
    
        #define __careful_cmp(op, x, y)					\
    	__builtin_choose_expr(__is_constexpr((x) - (y)),	\
    		__cmp(op, x, y), __careful_cmp_once(op, x, y))
```

Depending on the expressions passed in, this means that `min3()` can end up generating a fair amount of code. Even if one expects a large expansion, though, the actual amount may lead to significant eyebrow elevation: the single line of code shown above expands to _47MB_ of preprocessor output. Bergmann explained this result this way: 

> It nests min() multiple levels deep with the use of min3(), and each one expands its argument 20 times times now (up from 6 back in linux-6.6). This gets 8000 expansions for each of the arguments, plus a lot of extra bits with each expansion. PFN_DOWN(MAXMEM) contributes a bit to the initial size as well. 

Kernel developers, as a rule, care deeply about efficiency; that is especially true when it comes to the time required to do a kernel build. So it is unsurprising that this problem attracted some attention once it came to light. 

#### Minimizing the problem

Lorenzo Stoakes [brought the issue](<https://lwn.net/ml/linux-kernel/c83c17bb-be75-4c67-979d-54eee38774c6@lucifer.local/>) to the linux-kernel mailing list, showing how the 6.7 changes had made compilation time worse. Laight [posted a patch series](<https://lwn.net/ml/all/23bdb6fc8d884ceebeb6e8b8653b8cfe@AcuMS.aculab.com>) one day later that attempted to mitigate the problem. That series improved compilation time, though not enough to completely make up for the build-time regressions seen. It also ended up provoking some warnings from the test bots, and some of the changes to the macros made some developers ([including Bergmann](<https://lwn.net/ml/all/24be8665-4717-4ee2-8a81-80fed5181736@app.fastmail.com>)) nervous; those macros have reached a level of subtlety that makes people reluctant to change them. Torvalds, too, [was uncomfortable](<https://lwn.net/ml/all/CAHk-=wh_+muDANgpjC6_31QMh4OnKEOgbZiD_MymHxAHRZRyqg@mail.gmail.com>) with some of the changes, but he also wondered if they were the right approach to take in the first place: 

> I do get the feeling that the problem came from us being much too clever with out min/max macros, and now this series is doubling down instead of saying "it wasn't really worth it". 

He later [suggested](<https://lwn.net/ml/all/CAHk-=whLcr11D28vu2NotZYn3GNH6BCTK57Zw4d4v7eut39z=g@mail.gmail.com>) simply reverting the 6.7 changes even though the previous code was ""stupid and limited and caused us to have to be more careful about types than was strictly necessary"" but, as Stoakes [pointed out](<https://lwn.net/ml/all/d48ce3b3-9173-4309-aae6-96be42327f97@lucifer.local>), a lot of code in the kernel has since come to depend on the new functionality that those changes added. Reverting them now would not be a straightforward task. 

So Torvalds decided to take a bit of a different approach after [observing](<https://lwn.net/ml/all/CAHk-=wjPr3b-=dshE6n3fM2Q0U3guT4reOoCZiBye_UMJ-qg1A@mail.gmail.com>) that many of the worse expansion cases were, in the end, relatively simple constant expressions. Rather than try to fix the existing complex macros, he just added a couple more with a familiar look to them: 

```
/*
         * Use these carefully: no type checking, and uses the arguments
         * multiple times. Use for obvious constants only.
         */
        #define CONST_MIN(a,b) ((a)<(b)?(a):(b))
        #define CONST_MAX(a,b) ((a)>(b)?(a):(b))
```

By the time these macros [landed in the mainline](<https://git.kernel.org/linus/3a7e02c040b13>) they had naturally gained just a little complexity (and new names): 

```
#define MIN_T(type,a,b) __cmp(min,(type)(a),(type)(b))
        #define MAX_T(type,a,b) __cmp(max,(type)(a),(type)(b))
```

He converted a number of the worst expansion cases to use the new macros just prior to the 6.11-rc1 release, then merged [a patch](<https://git.kernel.org/linus/017fa3e89187>) taking away the ability for `min()` and `max()` to work as part of a constant expression. That simplified the code somewhat at the cost of making the macros unsuitable for use in places where constants are needed, but the new macros can be used instead in such situations. 

These changes will not entirely resolve the problem in cases where the expressions are not constant, so chances are that more tweaks to the regular `min()` and `max()` macros are in store. Meanwhile, though, we have had a convincing demonstration of the sorts of pitfalls that can accompany this sort of extensive use of the C preprocessor. It can accomplish some magical-seeming effects, but spells of this nature often have subtle and unpleasant side effects.
