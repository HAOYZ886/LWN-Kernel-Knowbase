---
title: The cost of inline functions
url: https://lwn.net/Articles/82495/
date: "April 28, 2004"
category: Coding style; Inline functions
author: 
---

The kernel makes heavy use of inline functions. In many cases, inline expansion of functions is necessary; some of these functions employ various sorts of assembly language trickery that must be part of the calling function. In many other cases, though, inline functions are used as a way of improving performance. The thinking is that, by eliminating the overhead of performing actual function calls, inline functions can make things go faster. 

The truth turns out not to be so simple. Consider, for example, [this patch](<https://lwn.net/Articles/82494/>) from Stephen Hemminger which removes the inline attribute from a set of functions for dealing with socket buffers ("SKBs", the structure used to represent network packets inside the kernel). Stephen ran some benchmarks after applying his patch; those benchmarks ran 3% _faster_ than they did with the functions being expanded inline. 

The problem with inline functions is that they replicate the function body every time they are called. Each use of an inline function thus makes the kernel executable bigger. A bigger executable means more cache misses, and that slows things down. The SKB functions are called in many places all over the networking code. Each one of those calls creates a new copy of the function; Denis Vlasenko recently [discovered](<https://lwn.net/Articles/82498/>) that many of them expand to over 100 bytes of code. The result is that, while many places in the kernel are calling the same function, each one is working with its own copy. And each copy takes space in the processor instruction cache. That cache usage hurts; each cache miss costs more than a function call. 

Thus, the kernel hackers are taking a harder look at inline function declarations than they used to. An inline function may seem like it should be faster, but that is not necessarily the case. The notion of a "time/space tradeoff" which is taught in many computer science classes turns out, often, to not hold in the real world. Many times, smaller is also faster.
