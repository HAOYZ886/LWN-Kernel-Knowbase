---
title: "How likely should likely() be?"
url: https://lwn.net/Articles/70473/
date: "February 10, 2004"
category: Coding style; likely
author: 
---

Newcomers to the kernel code base are often surprised by the appearance of (what seems to be) a bunch of calls to functions called `likely()` and `unlikely()`. These calls always appear in conditional tests, along these lines: 

```
if (likely(some_condition)) {
    	/* Do something */
        }
```

In fact, `likely()` and `unlikely()` are not function calls at all; instead, they are hints to the compiler. If the compiler knows that one outcome is far more likely than the other, it can optimize the code it generates accordingly. On some architectures, this information can also be encoded into the object code, where it will override the branch prediction normally done by the processor. 

David Woodhouse [noted](<https://lwn.net/Articles/70474/>) that the differing interpretation of these directives by different architectures makes it hard to know when `likely()` and `unlikely()` should be used. If the result of one of those directives is just a bit of code optimization, they should be used liberally whenever the programmer knows that one outcome will happen more often than the other. On some architectures, however, the cost of guessing wrong is fairly high, and these directives should only be used where the odds are overwhelmingly in favor of one outcome. 

David's proposal is to replace `likely()` and `unlikely()` with a new `probable()` macro: 

```
probable(condition, percent)
```

Where "`percent`" is the programmer's estimation of how often the condition will evaluate true. Each architecture could then decide what to tell the compiler based on the given percentage. 

Rusty Russell has [a more straightforward answer](<https://lwn.net/Articles/70476/>), saying that these directives should be rarely used. 

Sometimes, unlikely()/likely() help code readability. But generally it should be considered the register keyword of the 2000's: if the case isn't ABSOLUTELY CRYSTAL CLEAR, or doesn't show up on benchmarks, disdain is appropriate. 

The "disdain" approach seems more likely to be adopted than a new macro. There will be very few code paths where these directives will make a measurable difference. And the fact is that programmers often guess wrong about which code paths will be taken how often. 

David would also like to add a probability to the `get_unaligned()` macro, which is used to access data which might not have the alignment required by the processor. Some architectures can handle any alignment; on those, `get_unaligned()` expands to a direct pointer dereference. Others require that unaligned access be done via multiple, smaller fetches or stores. Of those, some architectures can fix up an unaligned access attempt in an exception handler, and others cannot. For architectures which can fix unaligned accesses, it might be faster to take an occasional exception if the probability of an unaligned access is small. Adding a probability to the `get_unaligned()` macro (and `put_unaligned()` as well) would allow each architecture to optimize those accesses. Whether the resulting performance improvement would justify the effort remains to be seen.
