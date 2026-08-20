---
title: The newest development model and 2.6.14
url: https://lwn.net/Articles/157951/
date: "November 2, 2005"
category: "Development model-Kernel quality"
author: 
---

The [2005 Kernel Summit](<https://lwn.net/Articles/KernelSummit2005/>) made some tweaks to the kernel development model with the aim of producing higher-quality releases in a more timely manner. To that end, it was said that major changes would only be allowed during the first two weeks of each development cycle; after that, only bug fixes could go in. The hope was that this rule would eliminate destabilizing patches late in the cycle and concentrate developers' minds on making things work. 

The 2.6.14 kernel is the first to go through the entire cycle since the kernel summit. This kernel, released on October 27, came out almost exactly two months after 2.6.13, which showed up on August 29\. That is relatively fast by 2.6 standards, but [still too slow](<https://lwn.net/Articles/157957/>) for some developers. The complainers feel that the freeze period puts too much of a damper on development, and that, somehow, the kernels should come out faster. 

2.6.14 would have come out sooner were it not for a final delay to fix some remaining bugs (some of which turned out not to be real). Linus, however, [is pretty happy](<https://lwn.net/Articles/157958/>) with how 2.6.14 worked. A number of significant changes were merged, but regressions in the released kernels seem to be within reasonable limits. As a result, Linus doesn't see the need to make further changes to the process at this time: 

So I'm planning on continuing with it unchanged for now. Two-week merge window until -rc1, and then another -rc kernel roughly every week until release. With the goal being 6 weeks, and 8 weeks being ok. 

Andrew Morton, meanwhile, [has an answer](<https://lwn.net/Articles/157960/>) for those who think the development cycle is still too long: 

a) you're sitting around feeling very very very bored while 

b) the kernel is in long freeze due to the lack of kernel developer attention to known bugs 

The solution seems fairly obvious to me? 

It was pointed out that many bugs relate to hardware which most developers do not have. The response was that sometimes developers have to talk to users who encounter bugs and try to track them down anyway. In any case, the ongoing effort to get developers to fix bugs seems likely to be necessary for some time to come. 

One other branch of the discussion, meanwhile, took on the question of whether the kernel has gotten too big. Prompted initially by Roman Zippel, Andrew Morton [did some compile tests](<https://lwn.net/Articles/158174/>) and came out with some disturbing numbers: the size of kernels with similar configurations went from about 600K (2.5.71) to over 800K (2.6.8). He also noted that the use of a current version of gcc adds almost 100K to the final kernel size when compared to gcc 2.95.4. Clearly, some serious inflation is going on somewhere. 

Except that it's not quite so clear. Adrian Bunk [demonstrated](<https://lwn.net/Articles/158175/>) that, by using the `-Os` compile option (which instructs gcc to optimize for size), current compilers can make kernels which are quite a bit smaller than those made with the old 2.95 release. The resulting discussion suggests that the kernel developers may try making `-Os` the default for kernel builds in the future. Fedora already builds its kernels this way. The interesting thing is that, in the past, kernels built with `-Os` have often performed as well as (or even better than) those optimized for speed. Cache effects have a huge impact on kernel performance, and a smaller kernel is more cache friendly. 

Compiler issues aside, there truly has been some growth in the kernel. Linus [is not surprised](<https://lwn.net/Articles/158177/>) by this: 

On the other hand, I do believe that bloat is a fact of life.... The fact is, we do do more, and we're more complex. Our VM is a _lot_ more complex, and our VFS layer has grown a lot due to all the support it has for situations that simply weren't an issue before. And even when not used, that support is there. 

Expect an increase in de-bloating work in the near future. In some areas, this work has been ongoing for a while - consider, for example, the effort to shrink the `sk_buff` structure used to represent packets in the networking subsystem. For a more extreme example, see [Matt Mackall's SLOB allocator](<http://lwn.net/Articles/157944/>), a replacement for the slab subsystem which is much smaller, but which does not perform as well on larger systems. SLOB is not for everybody (it's mainly intended for embedded systems), but it almost certainly foreshadows a surge in Linux weight reduction patches.
