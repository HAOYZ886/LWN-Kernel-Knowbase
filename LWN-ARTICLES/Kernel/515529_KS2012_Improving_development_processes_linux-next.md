---
title: "KS2012: Improving development processes: linux-next"
url: https://lwn.net/Articles/515529/
date: "September 12, 2012"
category: "Development model-linux-next; linux-next"
author: "By Michael Kerrisk September 12, 2012 2012 Kernel Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Michael Kerrisk**  
September 12, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

The final session of day one of the 2012 Kernel Summit considered the `linux-next` tree and a possible complementary tree. 

Steven Rostedt stated that he'd like to have a "linux-devel" tree, which would serve a similar purpose to that once served by Andrew Morton's "-mm" tree: it would be a place where reasonably stable code sits for a while for longer testing. He noted that such a tree might be useful for an API that hasn't yet stabilized, for example. Steven asked whether others would also be interested in something like this. 

Chris Mason questioned whether such a tree could work in practice. ""When your work and my work are together, people blame me for your bugs and vice versa."" Based on experience with a similar approach in another project, Ben Herrenschmidt noted another problem: people started developing against that code base instead of the designated development base (i.e., the creation of a "linux-devel" might cause some people to develop against that tree instead of `linux-next`). Tony Luck noted that the value of a "linux-devel" tree would depend greatly on how much testing it received, and the sense was that such a tree would likely see less testing than `linux-next`, which itself could do with more testers. 

Of course, even if a "linux-devel" tree was considered worthwhile, the tree would need a maintainer. In response to the question of how much work was required to maintain `linux-next`, the maintainer, Stephen Rothwell, said it required between four and ten hours per day, depending on the stage in the kernel-release cycle. In the end, as Steven Rostedt himself noted, the overall response to the proposal of a "linux-devel" tree was unenthusiastic. 

Attention then briefly turned to the `linux-next` tree. Ted Ts'o asked: are people happy with how the tree was working? The overall consensus seemed to be that it was working well. H. Peter Anvin seemed to sum up the mood, in stating his overall contentment with `linux-next` while noting that ""the imperfections of `linux-next` are reflections of the fact that it is a real-world creation"". Ted asked in a tone that seemed to expect a negative answer, ""does anyone run `linux-next` in anger on their development system?"", and was a little surprised to see that quite a number of kernel developers indicated that they do eat their own dog food, living pretty much continuously on `linux-next` as the booted kernel on the work system that they use on a daily basis. After more than three years, it's clear that `linux-next` is by now an essential part of the kernel-development model.
