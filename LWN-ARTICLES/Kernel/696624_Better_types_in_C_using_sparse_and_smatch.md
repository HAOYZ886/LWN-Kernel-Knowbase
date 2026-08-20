---
title: Better types in C using sparse and smatch
url: https://lwn.net/Articles/696624/
date: "August 10, 2016"
category: Smatch
author: "August 10, 2016 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

August 10, 2016

This article was contributed by Neil Brown

The primary motivation for my recent examinations of [sparse](<https://lwn.net/Articles/689907/>) and [smatch](<https://lwn.net/Articles/691882/>) came from a fascination with the idea that they can be used to make a better, safer version of C. They cannot be used to make it easier to write good programs, but they can make it harder to write bad programs by detecting constructs that are unwanted even though they are not errors in true C.

Sparse already provides for `address_space` and `bitwise` annotations on pointers and integers, respectively, ensuring that types the programmer wants to keep distinct can be kept distinct. Motivated by this existing functionality, and a particular need of my own, I set out to discover if either sparse and smatch (or both) could be used to keep track of which pointers might be null and to warn about any code that could lead to a null pointer being dereferenced. Though I cannot yet declare complete success, the results have been fairly encouraging and distinctly educational. In the interests of sharing this education, the current state of success and failure is presented below.

#### Preliminary observations

Dereferencing null pointers in C is far from a new concern, so it would be surprising if there was nothing already available to address this concern; a quick scan of the GCC documentation reveals that it already has a "`nonnull`" attribute for functions. The example in [the documentation](<https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#Common-Function-Attributes>) shows:

```
extern void *my_memcpy(void *dest, const void *src, size_t len)
                  		  __attribute__((nonnull (1, 2)));
```

This declaration tells the compiler that the first and second argument will never be null. Further examination shows that this is not useful for my purposes as it facilitates optimizations more than warnings. The compiler is free to remove any code in `my_memcpy()` that would only be run if one of those pointers were null, and it _may_ sometimes warn if a null value is passed as an argument. Since it provides no certainty of warning and only applies to function arguments and not, for example, structure fields, I find it of little use.

My particular use case is the [editor-building framework project](<http://git.neil.brown.name/?p=edlib.git;a=summary>) that I [spoke about at linux.conf.au in January [video]](<http://youtu.be/kjfHEGeqpIY>), which currently contains about 18,000 lines of C code. I started out, as in many projects, not really being sure how I wanted various aspects to work. As the project matured, I realized that there were a great many places where I had assumed pointers would be non-null, but where I really should check. This doesn't apply to all pointers; some, by design, must never be null. Others merely _should_ never be null, so checking is indicated. I could audit all that code manually, but I would much rather have a tool to help me.

Looking more closely at the tools at hand, I discovered that sparse knows about a rarely used "`safe`" attribute that is meant for "non-null/non-trapping pointers". If a variable is declared to be `safe` as, for example, in:

```
char *p __attribute__((safe));
```

then any attempt to test whether the value of that variable is (or is not) null produces a warning. While this functionality is not, by itself, hugely useful, the fact that sparse already parses and stores the annotation is; it provides a basis on which to build.

A few moments thought are enough to determine that, while it must always be safe to dereference a `safe` variable, it does not follow that it is always unsafe to dereference other variables. As a trivial example:

```
if (p)
    	*p = 0;
```

must always be safe, at least against dereferencing a null pointer. This sort of dependency is not something that sparse is able to resolve, but it is exactly the sort of thing that smatch was built to handle. 

As smatch was built on sparse, it has access to the `safe` attribute too, though it doesn't keep track of attributes quite as well as sparse and needs some coaxing. Once this attribute is tracked properly, smatch should be able to know when a variable is safe, either because it was annotated as being safe, or because its value has recently been tested and found to be non-null. As we found in my recent analysis, it is quite easy to extend smatch with a new checker, so that seemed like a profitable course to follow. 

#### Building a checker for safe pointer dereferencing

Building a new checker for smatch is quite easy, though I must thank Dan Carpenter for providing me with an early example to work from. That example has since been discarded and rebuilt from scratch, but the knowledge gained was invaluable. A sanitized development history of my checker can be seen on [GitHub](<https://github.com/neilbrown/smatch/compare/06e59b03b...14646fee3>) with the [first revision](<https://github.com/neilbrown/smatch/commit/833066fdf55c>) limited to reporting all the places in the code where the `DEREF_HOOK` is called. As this checker will eventually expect to find `safe` annotations and so will complain extensively about any program that isn't appropriately annotated, the checker will only activate if `SMATCH_CHECK_SAFE` is set in the environment. With this environment variable set, the enhanced smatch can be run on any C program and will report all the places were a pointer dereference is found. Somewhat surprisingly, it reports on a lot more too.

In most of the computer programming world, the term "dereference" is [reserved for pointers](<https://en.wikipedia.org/wiki/Dereference_operator>). A "reference" is another name for a "pointer", and when code accesses the memory pointed to, it is said to be "dereferencing" that pointer. However, in sparse, the term `DEREF` — or more specifically `EXPR_DEREF` — refers to the operation of accessing a member within a structure, that is the dot ("`.`") operator. So a construct like `a->b` is converted to `(*a).b` and parsed as: 

```
EXPR_DEREF( EXPR_PREOP('*', EXPR_SYMBOL('a')), 'b')
```

so dereferencing is a `*` prefix operation, and the dot operator is called `EXPR_DEREF`. Since sparse uses this terminology, it makes some sense for smatch to use it too, so `DEREF_HOOK` hooks fire both for member access and for real pointer dereference with the `*` operator. Once this is understood, it is easy to [only consider](<https://github.com/neilbrown/smatch/commit/84fea8bc43da>) `DEREF_HOOK` calls when an `EXPR_PREOP` expression is given.

With this more proper accounting, my project reports 7104 dereference operations — some of which I know to be unsafe, most of which I hope are safe and that I want the checker to confirm are safe. Now that the prototype checker is finding the target expressions, the `implied_not_equal()` interface provided by smatch can be used to start ignoring dereferences that can be determined to be safe. [Adding that call](<https://github.com/neilbrown/smatch/commit/e92c259f9c93>) reduces the number of dereferences reported to 1643. This large drop might seem to suggest that I had already been quite careful but, alas, this is not the case. When smatch notices that a pointer has been dereferenced, it [records](<https://github.com/neilbrown/smatch/blob/2016-aug/smatch_extra.c#L1083>) that it must now have a value in the range for valid pointers. This means that subsequent dereferencing on the same value will notice that the value is certainly not `NULL`. So a large part of this drop is just removing noise rather than detecting known-safe usage.

The next step involves adding a large number of `__attribute__((safe))` annotations and [updating the code to check for these](<https://github.com/neilbrown/smatch/commit/b3a18cf62329>). The word `safe` currently appears 871 times in my code, so this was not a trivial task, but as I had a tool to help me find places where it was needed, it was largely a mechanical one. Here the use of sparse in parallel with smatch was particularly useful. Though smatch shares much code with sparse, it does not perform all the same tests. In particular it doesn't complain if a `safe` value is tested, and doesn't complain if a function declaration uses different annotations from the function definition. Using sparse, I could be sure that functions were declared consistently and would often be warned when I declared something as `safe` that I probably shouldn't have.

Actually adding the text `__attribute__((safe))` throughout the project would have resulted in extremely ugly code, but that is just the sort of problem that the C pre-processor turns into a non-problem:

```
#ifdef __CHECKER__
        #define safe __attribute__((safe))
        #else
        #define safe
        #endif
```

Now I just use the simple word `safe`. e.g.

```
struct pane *focus safe;
```

With lots of annotations and a version of my checker that ignores `safe` values, I had reduced the number of interesting pointer dereferences down to 786; still too many, but there was still some low-hanging fruit to be removed. One pattern that showed up repeatedly when adding `safe` annotations was that a `safe` value, possibly from a function parameter or a structure member, would be assigned to a local variable, and then the local variable would be dereferenced. Marking that local variable as `safe` seemed excessive; tracking this sort of status is exactly what smatch is good for.

After [a little code rearrangement](<https://github.com/neilbrown/smatch/commit/4a41fa893f82>), a [new hook](<https://github.com/neilbrown/smatch/commit/cf1abbea88e5>) was added to process all assignments and to mark the variable on the left as `safe` if the value on the right was known to be non-null. As with dereferences, we need to be selective about which assignments are considered: assignments like "`+=`" will never change the `safe` status of the left-hand-side, so only simple "`=`" assignments need to be considered. The easiest way to mark a variable as `safe` is to define a smatch state and associate that with the left-hand expression, and to be sure to remove it when there is the possibility of a null value being assigned. Doing this brings the number of interesting dereferences down to 374.

We are now using two distinct states to record that a variable may be safe to reference: the new "`safe`" state that is assigned when a value is assigned with a safe value, and the numeric-range state that is maintained internally by smatch. This causes a little confusion when the two need to be merged. For example in the code fragment:

```
if (!p)
    	p = safe_pointer;
        *p = 0;
```

For the case where `p` was originally null, the checker will mark `p` with the `safe` state when `safe_pointer` is assigned to it. For the case where `p` was not null, smatch will record this fact in its numeric-range state. When the code `*p = 0` is reached, those two states will not have been merged as they are incompatible. Instead, the checker would need to examine the tree of historical states (described in the previous smatch article) and ensure that each branch is safe. This issue doesn't affect many cases in my code and so hasn't been addressed yet.

Once we have the option of marking variables, fields, functions, and function parameters as `safe`, we have introduced new places where errors can occur: only safe values may be assigned to, returned from, or passed into these various places. Given the infrastructure we already have, these checks can be [added to the assignment hook](<https://github.com/neilbrown/smatch/commit/4c03b3771adf>), to [a new function call hook](<https://github.com/neilbrown/smatch/commit/7b1d4a18f51a>), and [to a return hook](<https://github.com/neilbrown/smatch/commit/c2345b2d4d4b>) with a minimum of fuss, though, as the return hook doesn't know the type of the function, it needs to pass information to the end-of-function hook.

These various checks add nearly 500 new warning sites and, while this sounds like a lot, it doesn't really add new classes of errors. A good number of these reports are the actual errors that I wanted to find, where I haven't been careful enough and want to be reminded that I should add proper checking. Most of the rest fit into one of a small number of categories, some of which can be addressed with improvements to the assessment of when a value is `safe`, but some that will require more major surgery to properly resolve.

#### Detecting more "safe" values

Handling pointer arithmetic is obviously necessary in order to handle array references, as these are translated to pointer addition early in the parsing process. Using the lower-order bits of a pointer (that would normally be zero) to store some flags or other data is a technique that should be familiar to most kernel programmers. A simple example of this is the "red-black tree" code which stores [the "color" of a node in the least significant bit of the parent pointer](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/rbtree.h?id=523d939ef98fd712632d93a5a2b588e477a7565e#n35>). The bit masking needed to extract a pointer, like the addition needed for arrays, needs to be recognized and handled by the dereference checker so that they don't cause it to lose track of which pointers are `safe`. This is not particularly hard, but requires more care than the other steps. [Adding this](<https://github.com/neilbrown/smatch/commit/78cca86f9c17>) reduces the number of possible null dereferences from 374 to 319.

A slight variation of pointer arithmetic is taking the address of a member of a structure. If `ptr` is a safe pointer to a structure containing the field `member`, then `&(ptr->member)` must be a safe pointer as well. Though such a construct will rarely be dereferenced directly, it will often be passed as an argument to a function. When trying to recognize a construct such as this within smatch, it is important to remember that the `expression` data structures used have not been completely normalized yet so, for example, parentheses and casts might still be present. Smatch provides [`strip_parens()`](<https://github.com/neilbrown/smatch/blob/2016-aug/smatch_helper.c#L556>) that will just remove any enclosing parentheses, and [`strip_expr()`](<https://github.com/neilbrown/smatch/blob/2016-aug/smatch_helper.c#L571>) that will also strip away casts and a few other constructs that are often uninteresting. Using these, an expression that finds the address of a structure member by way of a dereferenced pointer [can be detected](<https://github.com/neilbrown/smatch/commit/14646fee3fa4>), and then the safety of that inner pointer assessed. Adding this check removed nearly 160 warnings about unsafe values being passed as function arguments.

Making allowances for code included from common header files is sometimes easy and sometimes challenging. If it is just a function declaration that needs some `safe` annotation, then just adding a new declaration to a local header files will often suffice:

```
char *strncat(char *s1 safe, char *s2 safe, int n) safe;
```

The Python C-API provides some interfaces as macros that will dereference pointers that the programmer cannot declare as `safe` without changing the installed header files. Smatch provides an easy way to see if some code came from a macro expansion, but doesn't make it easy to tell if that macro was defined in a system include file — and so could be treated leniently — or in a local file — and so should be treated strictly. [Adding a check for macros](<https://github.com/neilbrown/smatch/commit/196d98fe0702>) and ignoring any dereference that came from them removes about 100 warnings from external macros, but, unfortunately, it also removes about 70 warnings from macros local to the package that should be treated more strictly.

#### A need for a richer type language

After the easy (and the not-quite-so-easy) mechanisms for tracking safe pointers have been dealt with, the remaining warnings are a fairly even mix of bugs that should be fixed and use cases that I know are safe for reasons that cannot be described with a simple `safe` annotation. These fit into two general classes.

First, there some structures in which certain fields are normally guaranteed to be non-null, but within specific regions of code — typically during initialization — they might be null. I really want two, or maybe more, variants of a particular structure type: one where various fields are `safe` and one where they aren't. Then, when using a pointer to the non-safe type in a context where the safe version is needed, the individual members could be analyzed and warning given if the members weren't as safe as they should be. More generally, this seems to fit the concept of a parameterized type where the one type can behave differently in different contexts. Allowing some attribute to apply to a structure in a way that affects members of the structure seems conceptually simple enough. Retro-fitting the parsing and processing of those attributes to sparse would be a more daunting task.

The second class is best typified by an extensible buffer like:

```
struct buf {
    	char *text;
    	unsigned int len;
        };
```

If `len` is zero, then `text` may be `NULL`. If `len` is not zero, then text will not be `NULL` (i.e. will be `safe`) and in fact will have `len` bytes allocated. I feel I want to write:

```
char * text __attribute__(("cond-safe",len > 0));
```

This is similar to a parameterized type except that the variation in type is caused by a value within the structure rather than an attribute or parameter imposed on the structure. This sort of construct is normally referred to as a "dependent type", as the type of one field is dependent on the value of another. I have no doubt that smatch could be taught to handle the extra dependency of these dependent types, providing that sparse could parse them and record the dependency properly.

Properly resolving these two would require a substantial effort and so is unlikely to happen quickly. As an alternative, the time-honored tradition in C of using a type cast to hide code that the compiler cannot verify can be used. If I have a pointer that I know to be safe, I can cast it to `(TYPE *safe)`, or, if I have a value that sparse thinks is safe but which I want to test anyway, I can test `(void *)safe_pointer`. With luck, this will allow all of the current warnings to be removed without too much ugliness.

#### Other possibilities

While I was working on this extension to smatch, the preliminary email discussions leading towards this year's Linux Kernel Summit were underway and Eric Biederman, quite independently, [started a discussion thread](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-July/002755.html>) titled "More useful types in the linux kernel" to explore the idea of strengthening the type system of C in order to benefit the development of the Linux kernel.

Biederman was initially thinking of a GCC plugin rather than enhancements to sparse, and his interest in pointer safety was more around whether appropriate locks and reference counts were held, rather than my simple question of whether the pointers are null or not. Stepping back from those details, though, the general idea seemed similar to my overall goal and it was pleasing to know that if this was a crazy idea I, at least, wasn't the only one to have it.

Subsequent discussion showed that, though [not everyone wants to run a time-consuming checker](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-July/002772.html>) every time they compile their code, many people would like to see more rigorous checks being applied. [One observation](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-July/002763.html>) that was particularly relevant to my work was that, in the kernel, pointers can have three different sorts of values: they can be valid, they can be null, or they can [store a small negative error code](<https://www.kernel.org/doc/htmldocs/kernel-hacking/convention-returns.html>). In the context of the kernel, just testing that a pointer is not zero is not enough to be sure it can safely be dereferenced.

There was even a [suggestion that a function declaration might explicitly list the possible error codes that might be returned](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-July/002843.html>), which would make for a much richer type annotation than the simple `safe` flag that I have been working with. Whether this sort of detail is really worth the effort is hard to know without trying. It may allow us to automatically catch a lot more errors and provide reliable API documentation, but it might — [as James Bottomley feared](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-July/002936.html>) — end up as ""a lot of pain, for what gain?""

As is often the case, abstract discussion is only of limited use. To find real answers we need to see real code and real results. When the required language extension is a single attribute that is already parsed by sparse, the exercise described here shows that getting those results is challenging but not prohibitive. For any more adventurous extensions, sparse would need to be be taught to parse more complex attributes and the difficulty of such a project is not one that I am able to estimate as yet. However we are a large community and there are clearly a few people interested. It is reasonable to hope that such extensions may yet be attempted and the results reported.
