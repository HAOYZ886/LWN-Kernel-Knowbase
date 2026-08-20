---
title: Fault injection and unexpected requirement injection
url: https://lwn.net/Articles/364742/
date: "December 2, 2009"
category: Development model
author: "By Jonathan Corbet December 2, 2009"
---

By **Jonathan Corbet**  
December 2, 2009

Good developers carefully write their code to handle error conditions which may arise. This code frequently suffers from one problem, though: test coverage is hard. Many of the anticipated errors never come about, so the error-handling code never gets exercised. So when things go wrong for real, recovery does not work as expected. For a few years, the Linux kernel has had a [fault injection framework](<http://lwn.net/Articles/209257/>) designed to help in the debugging of some types of error-handling code. By forcing specific things (memory allocations in particular) to go wrong, the fault injection framework can help developers ensure that errors are really handled as expected. 

Sripathi Kodi recently posted [a patch](<http://lwn.net/Articles/364478/>) adding certain types of futex failures to the fault injection framework. Ingo Molnar [responded](<https://lwn.net/Articles/364744/>) with a potentially surprising request: 

Instead of this unacceptably ugly and special-purpose debugfs interface, please extend perf events to allow event injection. Some other places in the kernel (which deal with rare events) want/need this capability too. 

This "unacceptably ugly" interface has existed as part of the fault injection framework since 2006, so it is a little surprising to hear, now, that it cannot be used. Ingo is firm about this point, though, and appears unwilling to back down. 

Extending perf events for fault injection might be the right long-term solution. But this situation highlights a trap for developers which certainly acts to make participation in the development process harder. In his travels, your editor has heard complaints from developers who set out to accomplish a specific task, only to be told that they must undertake a much larger cleanup to get their code merged. The topic also came up at the 2009 kernel summit; there, the consensus seemed to be that this kind of request can quickly become unreasonable. 

In this case, Sripathi has not been asked to fix the remainder of the fault injection framework code. But adding a new functionality to the perf events subsystem still likely goes rather beyond the scope of the original project. Sripathi has not responded to this request, so it's not clear whether we'll see a futex fault injection mechanism reworked to fit the new requirements, or whether this code will just fade away.
