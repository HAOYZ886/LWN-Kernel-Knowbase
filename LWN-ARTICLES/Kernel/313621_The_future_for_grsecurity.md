---
title: The future for grsecurity
url: https://lwn.net/Articles/313621/
date: "January 7, 2009"
category: "Security-Security technologies; grsecurity"
author: "By Jake Edge January 7, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
January 7, 2009

Using an out-of-tree kernel patch has several downsides but, as long as the patch is maintained and updated with the kernel, it is workable. If the developers lose interest--or funding--it suddenly becomes a much bigger problem for users. That scenario may be about to play out for users of the [grsecurity](<http://www.grsecurity.net/>) tool as a [recent release](<http://www.grsecurity.net/news.php#grsec2112>) comes with a warning that it could be the last. 

Users of grsecurity are, unsurprisingly, worried about the future of the security tool, but calls for its inclusion in the mainline are not likely to be successful. Over time, largely because of the efforts of others outside of the grsecurity project, various pieces of grsecurity (and the associated [PaX project](<http://pax.grsecurity.net/>)) have been added to the kernel. But, there are a number of reasons that the full grsecurity patch is not in the mainline; the most basic is that the developers seem unwilling or uninterested in following the normal path to inclusion. 

The grsecurity patch implements a number of security features that are useful, particularly for web servers or servers that provide shell access to untrusted users. One of the major features is [role-based access control](<http://en.wikipedia.org/wiki/Role-based_access_control>) (RBAC), which is an alternative to the traditional UNIX [discretionary access control](<http://en.wikipedia.org/wiki/Discretionary_access_control>) (DAC) or the more recent [mandatory access control](<http://en.wikipedia.org/wiki/Mandatory_access_control>) (MAC) provided by SELinux and Smack. The aim of RBAC is create a "least privilege" system, where users and processes have only the minimum necessary privilege to accomplish their task. grsecurity also includes hardening of the `chroot()` system call, to eliminate privilege escalation and other vulnerabilities from within a "chroot jail". In addition, there are a number of other miscellaneous features like auditing and restricting `/proc` information, all of which are [listed on the grsecurity features page](<http://www.grsecurity.net/features.php>). 

Another major component of grsecurity is the PaX code, which restricts memory use so that various exploits, such as buffer overflows and other code execution vulnerabilities, are blunted or eliminated. It does this by making data pages non-executable using--or emulating--the "no execute" (or NX) bit. PaX restricts `mprotect()` to not allow pages that are both writable and executable to avoid code injection as well. PaX also adds much more aggressive address space layout randomization (ASLR) than is currently used by Linux. PaX is developed separately from grsecurity, by the anonymous "PaX Team", then incorporated into grsecurity by developer Brad Spengler. 

The project has been around for a long time; grsecurity started in 2001, while PaX began in 2000. There are numerous satisfied users and grsecurity has been used in distributions such as [NetSecL](<http://netsecl.com/>) and [Hardened Gentoo](<http://www.gentoo.org/proj/en/hardened/>), but it has never made it into the mainline. Gabor Micsko recently posted a [request](<https://lwn.net/Articles/313758/>) on linux-kernel for Linus Torvalds to reconsider grsecurity: 

The common opinion of the developers of grsecurity, PaX and their users is that acceptance of the code into the kernel would be the best solution for saving the project, beside finding another long-term sponsor. 

Torvalds [replied](<https://lwn.net/Articles/313765/>) that much of what was in grsecurity and PaX was ""insane and very annoying and invasive code"". He then went on to explain some of the history: 

The apparent inability (and perhaps more importantly - total unwilling[n]ess) from the PaX team to be able to see what makes sense in a long-term general kernel and what does not, and split things up and try to push the sensible things up (and know which things are too ugly or too specialized to make sense), caused many PaX features to never be merged. 

Much of it did get merged over the years (mostly because some people spent the time to separate things out), but no, we're not going to suddenly start merging code like that just because the project is in trouble. None of the basic issues have been solved. 

A perfect example of the unwillingness to work with the kernel hackers is embodied in the decision [not to implement RBAC as a Linux Security Module](<http://www.grsecurity.net/lsm.php>) (LSM). For better or worse, LSM is the mechanism used to implement access control in the kernel. Conceptually, it is a good fit for the grsecurity RBAC code. It might require additional LSM hooks, but working on getting those hooks added is the right approach. There was some uncertainty about LSM at one time, but it clearly is the way forward today. 

There may also be an issue with the PaX code, in that anonymous contributions to the kernel are not allowed. Presumably Spengler, or some other interested hacker, could sign off on that code, but it cannot be accepted directly from "PaX Team". 

To the extent grsecurity and PaX have been proposed for inclusion, they have always been presented as a single monolithic patch. There has never been an attempt to break the patch up into logical chunks that can be accepted or rejected on their individual merits. So far, that has not occurred even after the project lost its sponsor. But waiting until the last minute is not going to work. As Robert Hancock [puts it](<https://lwn.net/Articles/313767/>): 

Saying to the kernel developers "here, throw this huge blob of code into your kernel because otherwise we're taking our ball and going home" is not how it works. 

If there is value in the existing code, interested users and developers need to work within the kernel process to get it accepted. To do that, one must identify the useful pieces and proceed from there. Valdis Kletnieks [suggests](<https://lwn.net/Articles/313766/>): 

Probably the best way to proceed would be for the stakeholders to come to some agreement on which parts are the "sane stuff" (which could be an interesting food fight), split those parts out, and submit them for inclusion as standalone separate patches. 

This is yet another example of the perils of out-of-tree code. By all accounts, there are satisfied grsecurity users who may well be left behind if the grsecurity project fails to find sponsors by the end of March. They can, of course, continue running the grsecurity-enhanced kernels they currently have, but may not be able to take advantage of upcoming kernel advances. 

Perhaps the stakeholders will gather together and continue updating grsecurity for newer kernels, but that still leaves the underlying problem. They would be better served spending at least part of their time working _with_ the kernel hackers to get as much of grsecurity and PaX as possible merged into the mainline.
