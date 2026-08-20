---
title: Better kernels with GCC plugins
url: https://lwn.net/Articles/461696/
date: "October 5, 2011"
category: "Build system-GCC plugins; GCC"
author: "By Jonathan Corbet October 5, 2011"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 5, 2011

Last week's [article on GCC plugins](<https://lwn.net/Articles/457543/>) drew [a comment](<https://lwn.net/Articles/461048/>) from "PaXTeam" regarding the PaX project's use of plugins. It seems they have developed a set of plugins to help with the task of creating a more secure kernel. Said plugins can be hard to find; your editor located them in [the grsecurity "test" patch set](<https://grsecurity.net/test.php>). There appear to be four of them, each of which tweaks the compilation process in a different way: 

  * Structures containing only function pointers are made `const`, regardless of whether they are declared that way. Of course, it turns out that this is the wrong thing to do in a number of cases, so the developers had to create a `no_const` attribute and use it some 180 places in their patch. 

  * A histogram of the distribution of sizes passed to `kalloc()` is generated; it's not clear (to your editor) what use is made of that information. 

  * Some fairly sophisticated tweaks to the generated assembly are made for AMD processors to improve the prevention of the execution of kernel data. 

  * Instrumentation is inserted to track kernel stack usage. 

Use of plugins in this way allows significant changes to be made to the kernel without actually having to change the code: 

To give you some numbers, an allmodconfig 2.6.39 i386 kernel loses over 7000 static (i.e., not runtime allocated) writable function pointers (a reduction of about 16%). creating an equivalent source patch would be thousands of lines of code and have virtually no chance to be accepted in any reasonable amount of time. 

On the other hand, plugins of this type can increase the distance between the code one sees and what is actually run in the kernel; it is easy to imagine that leading to some real developer confusion at some point. Still, says PaXTeam, ""the cost/benefit ratio of the plugin approach is excellent and there's a lot more in the pipeline"". It is not too hard to imagine other uses that are not necessarily tied to security. 

(Amusingly, the plugins are licensed under GPLv2, meaning that they do not qualify for the GCC runtime library exemption. The kernel does not need that library, though, so all is well.)
