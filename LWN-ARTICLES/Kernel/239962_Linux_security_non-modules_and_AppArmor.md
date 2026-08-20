---
title: "Linux security non-modules and AppArmor"
url: https://lwn.net/Articles/239962/
date: "June 27, 2007"
category: "AppArmor; Security-Security modules"
author: 
---

Long-time LWN readers will know that the Linux security module (LSM) API is controversial at best. To many, it has failed in its purpose, which is enabling the development of competing approaches to hardened Linux system; the only significant in-tree security module remains SELinux. Meanwhile, the LSM interface is easily abused; since it allows the insertion of hooks into almost any system operation of interest, it can be used by other modules to provide non-security functionality. The LSM symbols are mostly exported GPL-only, but it is still possible for binary-only modules to abuse the LSM operations - and, apparently, some have done so. 

SELinux hacker James Morris has been pondering this issue recently; he has also noticed that the in-tree security modules (SELinux and the small module implementing capabilities) cannot be unloaded. So, he asked, why implement a modular interface at all? He has posted [a patch](<http://lwn.net/Articles/239697/>) which turns LSM into a static API with no exported symbols. With this patch applied, any needed security "modules" must be built into the kernel; there is no longer any way to add them at run time. 

There have been a few complaints, but, from your editor's point of view, it does not seem like anybody has come up with a compelling reason why it must be possible to unload security modules. Instead, it has been pointed out that maintaining a coherent security state in the presence of unloadable modules is nearly impossible. So this patch would appear to have reasonably good chances of being applied. The only question, perhaps, is whether the developers feel the need to provide an extended warning period for developers and users of out-of-tree security modules. 

One such module is AppArmor - the GPL-licensed security mechanism distributed by Novell. AppArmor has remained out of the tree for a long time while its developers have tried to address the various comments which have been posted over the years. A [new AppArmor patch](<http://lwn.net/Articles/239951/>) has been posted; many things have been fixed, but one of the core points remains: AppArmor still uses a pathname-based mechanism for its policy enforcement. This approach sits poorly with developers - especially those in the SELinux camp - who think that pathnames are an inherently insecure method. In their view, the only truly secure way to control access to objects is to put labels on the objects themselves. 

It seemed that this dispute had been resolved at the 2006 kernel summit, where it was determined that the use of pathnames was not enough to keep AppArmor out of the kernel. That has not stopped people from complaining, though, and those complaints redoubled when another pathname-based approach (TOMOYO Linux) was posted recently. If AppArmor does get into the mainline, it will have to be over the objections of developers who feel that is providing false security to its users. 

Andrew Morton appears to want to resolve this issue and get it off the mailing lists; he [sees two alternatives](<https://lwn.net/Articles/239964/>): 

a) set aside the technical issues and grudgingly merge this stuff as a service to Suse and to their users (both of which entities are very important to us) and leave it all as an object lesson in how-not-to-develop-kernel-features. [...] 

b) leave it out and require that Suse wear the permanent cost and quality impact of maintaining it out-of-tree. It will still be an object lesson in how-not-to-develop-kernel-features. 

It seems that Andrew would rather not be in the position of delivering object lessons on how not to develop kernel code by whatever means; he concludes with this request: 

Sigh. Please don't put us in this position again. Get stuff upstream before shipping it to customers, OK? It ain't rocket science. 

At the 2006 summit, Linus took a clear position that the use of pathnames for security policies seemed reasonable to him. Given that, along with the fact that AppArmor is being widely distributed, and it seems that, sooner or later, this module should find a home in the mainline - even if it is no longer in modular form.
