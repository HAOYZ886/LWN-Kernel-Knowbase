---
title: Better tools for kernel developers
url: https://lwn.net/Articles/811528/
date: "February 6, 2020"
category: Development tools
author: "By Jonathan Corbet February 6, 2020"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 6, 2020

By many accounts, the kernel project uses outdated tooling, far behind the state of the art that Kids Today tend to favor. The kernel's workflow has worked well (enough) for years, but there are signs that it may not be sustainable indefinitely. As a result, there has been [an ongoing conversation](<https://lwn.net/Articles/803619/>) about improving the kernel's workflow, but little has changed so far. The [posting of a simple tool called `get-lore-mbox`](<https://lwn.net/ml/workflows/20200201030105.k6akvbjpmlpcuiky@chatter.i7.local/>) is a sign that the rate of change may be about to increase. 

The kernel project's reliance on email strikes many as quaint and antiquated. It may indeed partly be a natural outcome of the aging nature of the kernel community; many of the developers there, especially in the important maintainer positions, got started well before tools like web-based Git forges existed. Indeed, some of them got started using punch cards and may still be unconvinced of the virtues of, say, text editors. But the truth of the matter is that there are a number of good reasons for the kernel community's continued reliance on email; there is little else that can handle a community of that size and diversity. 

So, while it seems that the future of email (as opposed to, say, proprietary services like Gmail) is uncertain at best, the path toward a replacement in the kernel community is unclear. Developers will have to be convinced that any new tools will make their lives better, not worse; busy maintainers have little patience for "improvements" that slow things down. 

The `get-lore-mbox` tool, posted by Konstantin Ryabitsev is not the product of a vast development effort; it adds up to about 500 lines of Python code. Its core function is simple: given the message ID for any email in a thread of interest, it will download the entire thread from the [lore.kernel.org](<https://lore.kernel.org/>) archive into a local mbox file. That alone can be useful for anybody who wants to catch up on an email discussion that may have flown by, but there are some other useful features: 

  * With the `-a` option, only the actual patch series at the heart of the thread will be downloaded; it will be sorted and stored in a form suitable for passing directly to [`git am`](<https://www.git-scm.com/docs/git-am/1.7.0>). Any useful tags posted as replies (Acked-by, for example) will be automatically added to the relevant patches. 
  * The `-t` option will cause tags posted in replies to the series cover letter to be applied to all patches in the series. 
  * If the thread contains multiple versions of a patch series, the latest one will be stored; the `-v` option can be used to request a specific version if need be. 
  * If no message ID is provided, it will read a message from the standard input and extract the ID from there, making single-keystroke operation possible in a number of mail readers. 

Kernel maintainers tend to spend a lot of time reading through email threads, organizing the patches found therein, applying tags, etc. The initial response to this tool (including on the private kernel.org users list) has made it clear that they have been wanting something like this for a long time. The developer time that will be saved far outweighs the time it takes to write a gadget like this, but nobody has ever created such a thing. 

One might well wonder why this tool didn't appear until now. It would seem that there were two things that needed to happen before this work could be done. One of those was the creation of a reliable archive for kernel mailing lists that could be easily queried by a program. Reliable archives have been scarce in general, and a single archive for all kernel-related lists didn't exist at all; without that, there is nothing for a tool like `get-lore-mbox` to obtain messages from. The lore.kernel.org archive has fixed that problem; it now seems like an indispensable part of the kernel workflow, but it's worth remembering that it is [less than two years old](<https://lwn.net/Articles/758034/>). 

Back in 1975, Fredrick Brooks wrote in [_The Mythical Man-Month_](<https://en.wikipedia.org/wiki/The_Mythical_Man-Month>) that an effective programming team should be organized like a surgical team, with a set of people filling specific roles. The kernel certainly has its surgeon at the top, complete with many of the traits that such professionals are known to display. Brooks said that every team should have two secretaries; the kernel community seems to be able to get by without those. But he also said that each team needs a toolsmith, focused on creating the tools that the team needs. 

The kernel project has long lacked a toolsmith. It doesn't even have somebody who can collect the tools created by maintainers for their own needs and make them suitable for more general use. As a result, maintainers often do without, or they use tools they threw together themselves when they really needed to be doing something else; that causes the community to lose a lot of opportunities for better productivity overall. 

One of the open secrets in the free-software world is that, while some areas of development are well supported by companies, others tend to languish. Both within the kernel and beyond it, there are many pressing needs that, it seems, are nobody's problem, so nobody is funded to work on them. The costs of this neglect are many: burned-out developers trying to keep projects going in their spare time, security vulnerabilities, missing tests and documentation, and a lack of tools needed by the community as a whole, among others. 

If only there were a successful industry consortium that could focus some attention (and resources) on problems of this nature. For the kernel there is one, in the form of the Linux Foundation, which has indeed supported the kernel community in a number of ways, including employing a few key developers and taking over maintenance of the kernel.org system after the 2011 compromise shone a spotlight on the consequences of neglect there. More recently, the Linux Foundation has supported the creation and operation of lore.kernel.org — and the development of `get-lore-mbox` as well. 

The kernel workflow discussions have made it clear, though, that there is a lot more work to be done in this area. The kernel's processes and tools have suffered from a lack of attention for many years; that is partly a result of the fact that, on the surface, things are working smoothly. The kernel community has managed to organize itself well and functions at a level that few other projects can match. When the kernel community has found the time to focus on tools — Git being the primary example — the effects have rippled across the development community as a whole. But reaching that point has required going through a number of crises when the kernel project's processes proved unable to support the community. 

It would be a great thing if the next big transition in the kernel's development process could happen without the "crisis" part. If sufficient effort and resources go into the tools that the kernel community will need in the coming years, that might just be possible. That suggests that the work being done at the Linux Foundation should not just continue, but grow into something bigger. Supporting this work (which requires getting its member companies to support it) is one of the best things that the Linux Foundation can do to support the kernel project as a whole.
