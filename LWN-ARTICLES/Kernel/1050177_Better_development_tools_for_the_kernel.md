---
title: Better development tools for the kernel
url: https://lwn.net/Articles/1050177/
date: "December 15, 2025"
category: "Development tools-b4"
author: "By Jonathan Corbet December 15, 2025 Maintainers Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 15, 2025

* * *

[Maintainers Summit](<https://lwn.net/Articles/1049982/>)

Despite depending heavily on tools, the kernel project often seems to under-invest in the development of those tools. There has been progress in that area, though. At the 2025 Maintainers Summit, Konstantin Ryabitsev, who is (among other things) the author of [b4](<https://b4.docs.kernel.org/en/latest/>), led a session on ways in which the kernel's tools could be improved to make the development process more efficient and accessible. 

He started with a plea for developers to let him know what is needed, since that is likely to work better than leaving him to figure it out on his own. He continues to work, slowly, on a `b4 review` command that would assist with the application of review tags to commits. He has also spent some time trying to integrate large language models (LLMs) with b4, without a huge amount of success. 

[![\[Konstantin
Ryabitsev\]](https://static.lwn.net/images/conf/2025/ossjp/KonstantinRyabitsev-sm.png)](<https://lwn.net/Articles/1050178/>) The LLM work is somewhat ironic, he said, since he has had to put a lot of time into protecting kernel.org from [scraper attacks](<https://lwn.net/Articles/1008897/>) run by companies seeking training material for their models. So he is simultaneously trying to make LLMs work while trying to block them from the site. On kernel.org, a number of services have been decoupled onto separate servers in an attempt to shield the lore archive from these attacks. He noted that the scrapers have started solving the challenges needed to get past [Anubis](<https://anubis.techaro.lol/>), so he has had to dial up the difficulty of those challenges. 

Kernel.org sends a lot of email, he said; that mail is often marked as spam at the receiving end even though he has jumped through all of the requisite hoops. The email that the kernel community generates is sufficiently different from the norm that it looks strange to a system that is increasingly focused on commercial email. Linus Torvalds suggested that the problem could be addressed by adding more emojis to patch postings. Ryabitsev, though, has become increasingly interested in solutions that deliver messages directly to lore, without sending it as email at all. Pieces of that puzzle are already in place; developers are using [lei](<https://lwn.net/Articles/878205/>) now to follow discussions without having to subscribe to mailing lists, for example. 

The systems behind kernel.org have been moved over to hosting at Akamai. He has been trying to keep kernel.org decentralized, with copies of the data behind kernel development widely distributed. If somebody wanted to take kernel.org off the net, he said, they likely would succeed, but developers, with local copies of everything they need, would be able to continue working. Still, more thought needs to go into how the project would continue if its provider goes out for an extended period. He wants to get to a point where developers can communicate even if lore is gone. 

He has also been working on a new ring of trust that is more robust than the current solutions; it is not ready yet. Torvalds noted that he left home for this meeting without all of the keys for developers he pulls from on his laptop, and those keys were not present in the kernel's key repository. He put out a plea for developers to ensure that Ryabitsev has their GPG keys so that he can pull from them. 

The kernel bugzilla server, Ryabitsev said, is ""semi-dead"", and has been for several years. He suggested that the time has come to simply get rid of it. That server is running bugzilla 5.2; upstream is up to 5.9, but there is no upgrade path to get there. If the bugzilla server is removed, he said, he would find a way to keep the existing history around, but it would not be possible to create new entries. There did not seem to be any opposition to removing the bugzilla server (which has never been all that extensively used in the kernel community), but it will not happen immediately. 

Patchwork, he said, is used extensively. He is working on getting it to use lei queries to see when specific files have been touched. Torvalds said that the emails he gets from the pull-request tracker arrive within five minutes, and he loves that, but email from patchwork can take days. He was wondering what was going on, but nobody seemed to have an answer. 

Jakub Kicinski suggested that it is time to move on from patchwork. Ryabitsev asked what the replacement would be; Kicinski responded that it should be possible to ""vibe-code something in a day"". Ted Ts'o said that he is ""utterly reliant"" on patchwork to keep track of the outstanding patches. He doesn't need patchwork specifically, though, as long as something provides patch tracking; a system that was integrated with lore would be nice. Ryabitsev said that some of that functionality could maybe be incorporated into public-inbox (the email archive system behind lore and [LWN's email archive](</ml/>)); the Linux Foundation has been sponsoring work in public-inbox for a while now. 

The session ended there; Ryabitsev said that he would post a summary of what was discussed. [That summary](<https://lwn.net/ml/all/20251209-roaring-hidden-alligator-068eea@lemur>) duly arrived shortly thereafter.
