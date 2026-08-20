---
title: ARM wrestling
url: https://lwn.net/Articles/437162/
date: "April 6, 2011"
category: "Architectures-Arm; Development model"
author: "By Jonathan Corbet April 6, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 6, 2011

The Linux kernel supports a wide variety of architectures, some of which are more prominent than others. The ARM architecture does not usually draw a lot of attention, but, over the years, it has become one of the most important architectures for Linux. There's now a vast array of embedded devices which run Linux because the kernel runs well on ARM. So when the mailing lists see extended and heated discussions about the quality of the ARM architecture code, it's worth paying attention. 

It all started early in the 2.6.39 merge window when Linus [objected](<https://lwn.net/Articles/437170/>) to one of many pull requests for an ARM subarchitecture. He complained about excessive churn in the architecture, duplicated code, board-specific data encoded in source files, and conflicts between different merge requests. Much of that board-specific data, he says, should be pulled out of the kernel and into the boot loader; others have suggested that device trees could solve much of that problem. Meanwhile, it is impossible to build a kernel which runs on a wide variety of ARM systems, and that, [he says](<https://lwn.net/Articles/437172/>), is a problem for the platform as a whole: 

Why? Think of the Ubuntu's etc of the world. If you can't make half-way generic install images, you can't have a reasonably generic distribution. And if you don't have that, then what happens to your developer situation? Most sane people won't touch it with a ten-foot pole, because the bother is simply not worth their time. 

There actually seems to be a bit of a consensus on what the sources of the problems with the ARM architecture are. The hardware itself varies widely from one chip to the next; each vendor's system-on-chip offerings are inconsistent with each other, and even more so with other vendors' products. According to Nicolas Pitre, the openness of Linux has helped to make ARM successful, but [is also part of the problem](<https://lwn.net/Articles/437184/>): 

On ARM you have no prepackaged "real" Windows. That let hardware people try things. So they do change the hardware platform all the time to gain some edge. And this is no problem for them because most of the time they have access to the OS source code and they modify it themselves directly. No wonder why Linux is so popular on ARM. I'm sure hardware designers really enjoy this freedom. 

So the ARM architecture is a massive collection of "subplatforms," each one of those subplatforms is managed independently, often by different developers, and few of those developers have the time for or interest in doing cross-platform architecture work. The result is a lot of code flux, many duplicated drivers, and lots of hacks. 

Complicating the situation is the simple fact that the kernel is a victim of its own success. For years developers have been beating on the embedded industry to work upstream and to get its code into the kernel. Now the industry is doing exactly that; the result is a _lot_ of code, not all of which is as nice as we would like. The fact that a lot of embedded vendors seem to have little long-term vision or interest in solving anything but immediate problems makes things worse. The result is code that "works for now," but which is heading toward a long-term maintenance disaster. 

How is this problem to be solved? It seems clear that the ARM architecture needs more maintainers who are concerned with cross-platform issues and improving the state of ARM support as a whole. There would appear to be a consensus that ARM maintainer Russell King is doing a good job with the core code, and there are a few people (Nicolas Pitre, Catalin Marinas, Tony Lindgren, etc.) who are trying to bring some order to the subplatform mess, but they seem to be unable to contain the problem. As [Nicolas](<https://lwn.net/Articles/437178/>) put it: 

So we need help! If core kernel people could get off their X86 stool and get down in the ARM mud to help sort out this mess that would be really nice (thanks tglx). Until then all that the few of us can do is to contain the flood and hope for the best, and so far things being as they are have still worked surprisingly well in practice for users.... 

And we can't count on vendor people doing this work. They are all busy porting the kernel to their next SOC version so they can win the next big Android hardware design, and doing so with our kernel quality standards is already quite a struggle for them. 

There are some developers who are willing to provide at least some of that help. The Linaro project could also conceivably take on a role here. But that still leaves open the question of just how the code can be cleaned up. Arnd Bergmann has [suggested](<https://lwn.net/Articles/437181/>) the radical step of creating a new ARM architecture tree with a new, clean, design, then moving support over to it. Eventually the older code would either fade away, or it would only be used to support older hardware. Creating a new architecture tree seems like a big step, but it has been done before - more than once. The x86-64 architecture was essentially a clean start from x86; the two platforms were then eventually merged back together into a much cleaner tree. PowerPC support went through a similar process. 

Whether that will happen with ARM remains to be seen; there are other developers who would rather perform incremental cleanups on the existing ARM tree. Either way, the first step will have to be finding developers who are willing to do the work. There is no shortage of developers who are interested in ARM, but fewer of them are willing and able to do high-level architectural work - and to deal with the inevitable resistance to change. As [Thomas Gleixner said](<https://lwn.net/Articles/437183/>): 

The only problem is to find a person, who is willing to do that, has enough experience, broad shoulders and a strong accepted voice. Not to talk about finding someone who is willing to pay a large enough compensation for pain and suffering. 

So there are some challenges to overcome. But there is also a great deal of economic value to the ARM platform, a lot of people working in that area, and a reasonably common understanding of where the problems are. So chances are good that some sort of solution will be found.
