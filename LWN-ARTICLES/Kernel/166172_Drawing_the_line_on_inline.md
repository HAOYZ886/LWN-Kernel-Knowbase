---
title: Drawing the line on inline
url: https://lwn.net/Articles/166172/
date: "January 3, 2006"
category: Coding style; Inline functions
author: 
---

Kernel programmers tend to like inline functions. They resemble C macros, in that they result in code inserted directly into the calling function, with no added function call overhead. But, unlike macros, they offer type checking and the ability to include multiple lines of code without adding a pile of backslashes. In cases where a function is optimized out entirely, an inline function turns into no code at all - a level of efficiency which is hard to beat. And, in some cases, inlining is required; consider, for example, functions which embody special assembly instructions needed by the kernel. 

Inline functions also have their costs, however. Their code is duplicated for every call, so inline functions which are called from more than one place make the kernel larger. Increasingly, developers are becoming aware that this size increase carries a performance penalty. As the gap between CPU and memory speeds grows, cache behavior increasingly determines how fast a program runs. So the performance benefits of inline functions are often, at best, illusory, and sometimes negative; a larger kernel will be a slower kernel. 

Ingo Molnar recently [raised this issue](<https://lwn.net/Articles/166182/>) with a set of patches changing how the kernel is built. By turning on unit-at-a-time compilation (which causes gcc to consider an entire file in its optimization decisions) and by turning off forced inlining, he was able to achieve a 5.3% size reduction. Taking things to an extreme, and applying these patches to an "allyesconfig" kernel (one with all configuration options turned on) [results](<https://lwn.net/Articles/166184/>) in a nearly 25% smaller kernel. That is, to say the least, a significant size reduction to be achieved by such a small patch. Anybody interested in de-bloating the kernel should be paying attention. 

These patches have not been accepted by everybody, however. In particular, the turning off of forced inlining is controversial. When gcc is not forced to honor the `inline` keyword, it makes its own decisions, based on the size of the function and how many times it is called. When told to optimize for size, in particular, gcc will have a strong bias against inline functions. This approach yields a significant size reduction, but there is a problem: Linus [doesn't trust the gcc maintainers](<https://lwn.net/Articles/166186/>) to code consistent and correct inline heuristics, and [Andrew Morton doesn't either](<https://lwn.net/Articles/166187/>). Rather than turning off forced inlining and letting gcc figure things out, they would rather go through the code and remove unnecessary inline declarations one by one. 

It is true that the kernel has been burned by changes to how gcc handles `inline` in the past. Since then, gcc seems to have gotten smarter, and one can argue that its maintainers have become more aware of the issues. There is also the little fact that cleaning up the existing inline declarations is not a small job; [Ingo says:](<https://lwn.net/Articles/166190/>)

There are 22,000+ inline functions in the kernel right now (inlined about a 100,000 times), and we'd have to change _thousands_ of them. They are causing an unjustified code bloat of somewhere around 20-30%. 

Arjan van de Ven [adds](<https://lwn.net/Articles/166192/>): 

The reality is, most driver writers (in fact kernel code writers) tend to overestimate the gain of inline in THEIR code, and to underestimate the cumulative cost of it. Despite what akpm says, I think gcc can make a better judgement than most of these authors (probably including me :). We can remove 6400 now, but a year from now, another 1000 have been added back again I bet. 

How all of this will turn out is unclear. Certainly one can expect a higher level of resistance to patches adding inline functions in the future. There is likely to be a long flurry of de-inlining patches as well. The ability to turn off forced inlining might be added to the build system as an experimental option; some distributors may even decide to use this option for the kernels they ship. But enough developers seem uncomfortable with the idea of turning off forced inlining wholesale that this option may not get beyond the "experimental" stage for some time.
