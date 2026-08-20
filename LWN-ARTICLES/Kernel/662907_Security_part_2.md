---
title: Security part 2
url: https://lwn.net/Articles/662907/
date: "November 4, 2015"
category: Security
author: "By Jonathan Corbet November 4, 2015 2015 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 4, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

On the middle day of the 2015 Kernel Summit, when mostly technical topics were considered, Kees Cook [urged kernel developers](<https://lwn.net/Articles/662219/>) to take security more seriously. The invitation-only group held a followup discussion one day later regarding what should actually be done to improve the kernel's security. It seems there is a possibility that we will see more security-related features going into the kernel in the near future. 

Ted Ts'o started by asking the group whether they were willing to add "painful security stuff" now. Kees suggested starting by taking another look at the [PaX](<https://pax.grsecurity.net/>) and [grsecurity](<https://grsecurity.net/>) patch sets. There are a lot of things that can be done, he said; it's mostly a matter of finding people to work on them. He is one of those people, but he's only one person and the problem is larger than that. 

James Bottomley asked if there was one security-related technology that the kernel absolutely should support. Kees answered that we should never execute user-space code in kernel mode. 

Linus said that it would indeed be useful for somebody to go through the PaX and grsecurity patches. But that code needs to be looked at carefully. For example, they have implemented protections against executing user-space code, but (according to Linus) they have done so badly. Now hardware offers that protection and we can use it; if we had accepted their code, we would be in a deep well we couldn't dig our way out of. He is glad he refused to take it. 

Other parts of those patch sets include "slow, horrible code" that affects fundamental parts of the kernel. He called out in particular the fact that grsecurity adds more use of the high-memory concept, while the kernel developers are instead trying to get rid of it. That said, he allowed that there is probably useful stuff to be found there. 

Kees said he would like to document the individual pieces of those patch sets, describing the class of bug addressed by each. The easiest to adopt, he thinks, would be the GCC plugin defending against integer overflows. That would bring the plugin infrastructure into the kernel process and allow developers to experiment with it. 

Greg Kroah-Hartman said that the [Core Infrastructure Initiative](<https://www.coreinfrastructure.org/>) (CII) is willing to fund work improving the security of the kernel, but there is nobody out there to actually do this work. What CII won't do is fund grsecurity to go off in a corner and do its own thing (and, he noted, grsecurity isn't asking for that). The problem is that people who can do this work tend to get quickly hired to do something else. 

Kees wants to start by overcoming the kernel community's cultural resistance to the addition of security technologies; he feels that some progress has been made in this area. After that, the next challenge is finding people to do the work. James asserted, though, that cultural resistance is necessary; security is a tradeoff, and we have to find the proper middle ground. So the argument will have to continue. 

Ted suggested the addition of kernel-configuration options for the more intrusive security lock-down features. We just have to accept, he said, that some of these features will be painful for developers and will often be disabled on developer machines. They are more than acceptable on handsets, though. It was noted that, in enterprise settings, there is often a need for extensive diagnostic information for support purposes. Security features tend to make such information unavailable, so they can't be enabled; given a choice between security and support, many companies will choose support. 

Kees said that support might be an area for some innovation. Kernel symbol information could be made available via an encrypted channel, for example, so it would be available to those who have the right key. James said that the checkpoint/restore work is an example of how things can be done. Initially it required the export of a lot of system information, but things have been closed down over time; features like the [`kcmp()` system call](<https://lwn.net/Articles/478111/>) allow checkpointing to be done without leaking important information. It should, he said, be possible to get debugging information out of the kernel without helping attackers. 

Ted highlighted live kernel patching as an interesting tradeoff; there is little observable difference between the application of a patch and the installation of a rootkit. So live patching may never be enabled in a number of environments. Masami Hiramatsu said that the module-signing mechanism could be used to verify kernel patches. 

The session ended with no clear conclusions. There does appear to have been a bit of a shift toward greater acceptance of security enhancements, but the real proof of that will come when the patches start circulating.
