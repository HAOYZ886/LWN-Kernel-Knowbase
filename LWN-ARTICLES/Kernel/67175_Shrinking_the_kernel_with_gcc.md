---
title: Shrinking the kernel with gcc
url: https://lwn.net/Articles/67175/
date: "January 21, 2004"
category: Build system; GCC; Optimization tools
author: 
---

It will come as no surprise to most Linux users that the kernel has grown over time. In general, the expansion in the kernel has been more than offset by the increasing power of the systems that it runs on, but there is still a price to be paid for kernel bloat. Extra memory has to be paid for, and other overhead - such as cache misses - can hurt the overall performance of the system. 

Andi Kleen has been putting some effort into making the kernel smaller through the use of some relatively new and obscure gcc options. He starts with `-Os`, as do most kernel shrinkers; this one simply tells the compiler to optimize for size rather than strictly for performance. Anecdotal evidence suggests that `-Os` not only produces a smaller kernel, but the resulting code also often runs faster as well. 

The next step was to [use `-funit-at-a-time`](<https://lwn.net/Articles/67176/>). This option is new; it will be part of the upcoming gcc 3.4 release. It causes the compiler to load the entire source file into memory before it begins generating code; the result is better inlining and dropping of unused functions. The result was a little over 3% reduction in kernel text size. The reasons for this shrinkage require further investigation; it may be that there is a significant amount of dead code in the kernel. 

Finally, Andi has also [enabled `-mregparm=3`](<https://lwn.net/Articles/66965/>), which instructs the compiler to pass up to three function arguments in registers, rather than on the stack. This option helps even more than `-funit-at-a-time`. Using all three options, Andi is able to reduce the text size by over 700KB. 

There is one potential problem with `-mregparm=3`, however: it changes the calling conventions within the kernel, and thus breaks binary modules. As one might imagine, some kernel developers are more worried about this than others. Red Hat kernel packager Arjan van de Ven has [stated](<https://lwn.net/Articles/67179/>) that he is using this option, and intends to build production kernels that way as well. As always, sympathy for the difficulties encountered by distributors of binary-only modules is low. If the kernel hackers decide that this option is worth using, they'll not let some broken binary modules stop them.
