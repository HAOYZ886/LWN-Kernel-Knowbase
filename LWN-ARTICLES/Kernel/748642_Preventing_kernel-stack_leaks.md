---
title: "Preventing kernel-stack leaks"
url: https://lwn.net/Articles/748642/
date: "March 7, 2018"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet March 7, 2018"
---

By **Jonathan Corbet**  
March 7, 2018

The kernel stack is a small, frequently reused region of memory in each thread's address space. That reuse allows for efficient memory use and good performance as a result of cache locality, but it also presents a problem: data left on the stack can also end up being reused in ways that were not intended. The PaX patch set contains a mechanism designed to clear that data from the stack and prevent leaks, but an attempt to merge that code into the kernel has run into a snag. 

By design, the C language does not define the contents of automatic variables — those that are created on the stack when the function defining them is called. If the programmer does not initialize automatic variables, they will thus contain garbage values; in particular, they will contain whatever happened to be left on the stack in the location where the variables are allocated. Failure to initialize these variables can, as a result, lead to a number of undesirable behaviors. Writing an uninitialized variable to user space will leak the data on the stack, which may be sensitive in one way or another. If the uninitialized value is used within the function, surprising results may ensue; if an attacker can find a way to control what will be left on the stack, they may be able to exploit this behavior to compromise the kernel. Both types of vulnerability have arisen in the kernel in the past and will certainly continue to pop up in the future. 

Note that, while most uses of uninitialized data can be squarely blamed on the programmer, that is not always the case. For example, structures stored on the stack may contain padding between fields, and the compiler may well decide that it need not initialize the padding, since the program will not use that memory. But that memory can still be exposed to user space, should the kernel write such a structure in response to a system call. 

The best solution to this problem would be to find and fix every location where on-stack variables are not properly initialized. Tools like [KASan](<https://lwn.net/Articles/612153/>) can help with this task, but chasing down this kind of problem is a never-ending game. It would, thus, be nice to have a way of automatically preventing this type of vulnerability. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

For some time, Alexander Popov has been working on a port of the [PaX](<https://pax.grsecurity.net/>) STACKLEAK feature to the mainline kernel; the [ninth version of the patch set](<https://lwn.net/Articles/748485/>) was posted on March 3\. This series adds a GCC plugin that tracks the maximum depth of the kernel stack; this information can be used to help prevent stack overruns. The main purpose of this tracking, though, is to allow the kernel to clear the kernel stack on return from every system call; the stack-clearing code can use the maximum depth to avoid clearing more stack space than was actually used. According to the cover letter, turning on this feature incurs a performance cost of about 1%; in return for this overhead, kernel code always runs in an environment where the contents of the stack are known to have been properly set. 

Incidentally, the "clearing" of the stack is not setting it to zero. Instead, a special poison value is used; that should help to identify crashes that are caused by the use of uninitialized on-stack variables. 

Kees Cook [remarked](<https://lwn.net/Articles/748646/>) that this series ""should be ready to land pretty soon"", but that was before [Linus Torvalds became aware of it](<https://lwn.net/Articles/748647/>). Torvalds was not pleased, and made it clear that the STACKLEAK code was unlikely to make it into the mainline in its current form. He complained that: 

It doesn't actually seem to help *find* bugs at all. As such, it's another "paper over and forget" thing that just adds fairly high overhead when it's enabled. 

He suggested that security developers should focus more on finding and fixing problems, thus improving the kernel, rather than papering over issues in this way. 

Needless to say, the developers involved see the situation a little differently. Cook [responded](<https://lwn.net/Articles/748648/>): 

I think it does improve the kernel, especially if we can gain more complete coverage through native compiler options (instead of just a plugin). Right now, for example, the kernel is littered with memset()s because the compiler can't be trusted to correctly zero-init padding, etc. This is an endless source of bugs, and this patch series provides a comprehensive and fast way to keep the stack cleared. 

That response led Torvalds to [start thinking about](<https://lwn.net/Articles/748649/>) what he described as a ""*smart*"" way of dealing with the problem. Simply clearing the stack did not strike him as *smart*, but having the compiler initialize all automatic variables to zero would be. This initialization would provide similar protection from uninitialized data, but it could also be omitted whenever the compiler could determine that the variable was properly initialized in some other way. The result should be protection with significantly lower overhead. 

That overhead could be reduced further in performance-sensitive code by adding a special marker for variables that the compiler should not initialize, even if it seems that initialization is necessary. Places where this marker is needed would stand out in performance profiles, and the marker itself would be a red flag that uninitialized data may be present. 

Cook was [in favor](<https://lwn.net/Articles/748650/>) of adding this functionality to the compiler, but he also said that it is insufficient. It takes a long time for a new compiler to be widely adopted; people will build new kernels with old compilers for [a surprisingly long time](<https://lwn.net/Articles/748074/>). So an approach based solely on the compiler will not provide anything close to universal coverage for years. Adding the stack clearing into the kernel can protect sites regardless of whether a new compiler is used to build it. He also pointed out that there are a couple of cases where the zeroing of automatic variables does not provide complete coverage. If a vulnerability allows an attacker to read data below the current stack boundary, it can be exploited to read the possibly interesting data that will be sitting there. Clearing the stack also wipes out data that might otherwise be read by an unrelated vulnerability, considerably narrowing the window in which that vulnerability could be exploited. 

The discussion has no definitive conclusions as of this writing. The STACKLEAK code has encountered a significant obstacle on its way into the mainline, but it shouldn't necessarily be written off quite yet. There do appear to be some valid reasons for having this feature in the kernel, in the short term at least, and the stack clearing can be disabled for users who do not want to pay the cost. So, with some persistence (and security developers have learned to be persistent), there may yet be a place for the STACKLEAK patches in the mainline.
