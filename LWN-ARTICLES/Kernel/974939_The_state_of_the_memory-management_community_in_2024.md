---
title: "The state of the memory-management community in 2024"
url: https://lwn.net/Articles/974939/
date: "May 28, 2024"
category: "Memory management-Development process"
author: "By Jonathan Corbet May 28, 2024 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 28, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

A longstanding tradition in the memory-management track of the [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) is a session with maintainer Andrew Morton to discuss the overall state of the community and the development process. The 2024 gathering upheld that tradition toward the end of the final day of the event. It seems that Morton and the assembled developers were all happy with how memory-management work is going, but there is always room for improvement. 

In 2022, Morton had [described](<https://lwn.net/Articles/894378/>) his (then) new plans for a Git-based patch-management scheme, involving separate mm-unstable and mm-stable trees for work in different stages of development. He started the 2024 session by saying that he had hoped that patches would move relatively quickly from the mm-unstable tree into mm-stable, but in reality it tends to take rather longer. As a result, his hopes that developers would be able to work against mm-stable have not really worked out. He is not sure how to make that aspect of the process work better. 

[![\[Andrew Morton\]](https://static.lwn.net/images/conf/2024/lsfmm/AndrewMorton-sm.png)](<https://lwn.net/Articles/974942/>) A recurring problem, he said, comes about when he is holding onto version 3 of a patch set, and the discussion has made it clear that a fourth revision is needed. In such cases, he asked, should he keep version 3 while waiting, or simply drop it? There is benefit in keeping patches in mm-unstable, even if they will eventually be superseded, because the higher visibility means that new problems and potential improvements continue to turn up. He asked for feedback, but the room was uncharacteristically silent in response. 

He observed that he does get a bit tired of asking developers to describe the user-visible effects of the bugs they are fixing (here's [an example](<https://lwn.net/ml/linux-kernel/20240523115624.d068dfb43afc067ed9307cfe@linux-foundation.org/>) that came by as this article was being written). He said that he always tries to make the request a little different but, in the end, asking that question has become what he does for a career. 

There is, he noted, [a new CVE process](<https://lwn.net/Articles/961978/>) for the kernel. He wondered who was evaluating potential CVE assignments for memory-management patches, and said that he would like for the memory-management community to help where they can. Security implications, in the end, are user-visible effects of bugs as well. Michal Hocko said that, if developers try to think about this aspect a bit more, they can start noting in their changelogs whether patches are security relevant. But, he said, most of the people in the room are not security experts; the obvious security issues can be called out, but developers often do not know when they are fixing a security bug. 

Brendan Jackman said that whether a given problem can be triggered from user space is valuable information to include in a changelog. Liam Howlett said that he had once fixed a use-after-free bug that, at the time, seemed benign, but it turned out that it could be exploited in combination with another bug. He had thought it couldn't be triggered from user space, but he was wrong and the result was a severe vulnerability. Jason Gunthorpe said that trying to score the security impact of bugs has legal implications, and he does not want to go anywhere near that. Saying that a bug is not exploitable is a scary claim to make, he added. Vlastimil Babka said that, even if a changelog notes that a bug can only be triggered by privileged users, that bug will still have a CVE number assigned to it. Jackman said that he would still like to know if capabilities are required to trigger a bug, though. 

Changing subjects, Hocko asked developers to refrain from sending new versions of their patches before the discussion on the previous version has completed. He also said that, once a patch lands in Morton's tree, the potential for significant changes drops. It is better, he said, to keep work out of that tree until there is some consensus that it is good. David Hildenbrand said that Morton will often pull in work just to see if it will compile; maybe there is a need for a separate "stabilizing" tree that is not fed into linux-next. The linux-next kernel, he said, still blows up too easily. Morton said maybe he should add an "mm-stupid" tree. 

Howlett said that he is being copied on patches far less frequently than before, and is missing work that he should have seen; he wondered if something has changed somewhere. Hocko suggested ensuring that the MAINTAINERS file is up-to-date. Morton said that he spends a lot of time ensuring that the right people see patches. Gunthorpe took a moment to suggest that more developers should be using the [lei tool](<https://people.kernel.org/monsieuricon/lore-lei-part-1-getting-started>). It can be used to subscribe to a file and will collect all of the patches that touch that file; there is no need to change the MAINTAINERS file to see them. It is even possible to subscribe to specific functions. 

Matthew Wilcox pointed out that there is a group of Rust developers trying to get their work in; that work includes things like wrappers for the `page` structure. What was the plan for getting that work into the mainline? Morton protested "but it's all in Rust!" More seriously, he said that this work should go upstream via the Rust tree. 

Wilcox also complained about [a series of header-splitting patches](<https://lwn.net/ml/linux-kernel/20240430152931.1137975-1-max.kellermann@ionos.com/>) that had been posted recently. The author, he said, is exclusively focused on reducing compilation time and does not understand the memory-management subsystem. These changes will make maintenance harder, and he would like to reject them. Morton said that he looked at some of those patches when they went by, saw that they were "broken", and has not looked at them since. 

The final topic was briefly raised by Wilcox, who noted that a number of kernel subsystems have moved to a group-maintainership model; he wondered if memory management needed to do that too. He was unsure that such a change would actually fix any problems. Morton said that, for all practical purposes, the subsystem is group-maintained now, even if he solely maintains the tree that eventually goes upstream. Wilcox closed the session by saying that he was happy with the process overall and thanking Morton for doing a great job.
