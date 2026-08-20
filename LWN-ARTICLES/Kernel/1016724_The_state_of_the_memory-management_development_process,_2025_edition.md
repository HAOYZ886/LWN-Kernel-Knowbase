---
title: "The state of the memory-management development process, 2025 edition"
url: https://lwn.net/Articles/1016724/
date: "April 14, 2025"
category: "Memory management-Development process"
author: "By Jonathan Corbet April 14, 2025 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 14, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Andrew Morton, the lead maintainer for the kernel's memory-management subsystem, tends to be quiet during the Linux Storage, Filesystem, Memory-Management, and BPF Summit, preferring to let the developers work things out on their own. That changes, though, when he leads the traditional development-process session in the memory-management track. At the 2025 gathering, this discussion covered a number of ways in which the process could be improved, but did not unearth any significant problems. 

Morton started with the ""usual whinges"", primarily that many memory-management changes take too long to finalize. He accepted responsibility for much of that problem, saying that he needs to send more emails to move things along. 

[![\[Andrew Morton\]](https://static.lwn.net/images/conf/2025/lsfmm/AndrewMorton-sm.png)](<https://lwn.net/Articles/1016725/>) He has heard complaints that he is overly eager to merge things, with the result that buggy code is fed into linux-next. Since memory-management bugs affect everybody, those complaints can come from far outside this subsystem. He has responded by being slower in accepting changes from submitters if he does not know them well, sending them a private note saying that he is waiting for reviews. But some unready code is still getting through, he said; he will seek to be more skeptical going forward. That said, problems are fairly rare, it is really a matter of fine tuning. 

Dan Williams (Morton said) has suggested that the mm-unstable branch, which holds the rawer code, not be fed into linux-next. The mm-stable branch, instead, is more fully cooked and almost never requires fixing. But, if mm-unstable is excluded from linux-next, the patches contained within it will not get wider testing, and that would eventually cause mm-stable to be less so. So he plans to create another branch, called mm-next, as an intermediate step. Then, mm-unstable (which could perhaps be renamed ""mm-crazy"") would be kept private. The problem with this approach is that it will add latency to the process if patches have to work their way through all three branches; that means that nothing posted after -rc5 would be ready for the merge window. Matthew Wilcox indicated that this result would be just fine with him. 

Morton said that some changes might go directly to mm-next, while mm-unstable will be the first stop for larger or unreviewed work. It will be his judgment call. One nice effect will be that he can keep piling changes into mm-unstable without breaking any of the linux-next rules — and also take them out. The plan had always been that code could be removed from mm-unstable if it proved to have problems, but that has not worked well in practice; there is a perception that, once Morton has accepted a patch, it is well on its way to the mainline. 

Lorenzo Stoakes said that the biggest problem for him is the uncertainty about what will happen with patches as they enter the process. He would like to see more rules laid down; perhaps patches would require a suitable Reviewed-by tag before they could move into mm-next, for example. 

A separate problem Morton raised is that nobody is reviewing David Hildenbrand's patches, that there are just too many of them. (My own feeling is that Hildenbrand tends to take on some of the hardest problems — [example](<https://lwn.net/Articles/1013649/>) — and that many people do not feel smart enough to review the results). Michal Hocko said that Morton's expectations regarding who will review specific patches are not always clear. There need to be developers with responsibility for all areas in the subsystem, not a single ""superman"" who handles everything. Morton should be asking for people to stand up and take responsibility, Hocko said. 

In general, Morton said, if anybody wants him to do something specific, they should just ask him. He will often get requests to hold off on merging a patch series for a while, for example, and he generally does so. Williams wondered why Morton should be finding reviewers for other developers' patches, though. Everybody should be asking themselves why they aren't reviewing some developers' work. 

Morton suggested that all developers should also be scanning the linux-mm list for things needing attention. Hocko said that he had been doing that, but he just doesn't have time anymore; keeping up with the list is not a sustainable activity. 

Wilcox asked about the problem of ""zombie maintainers"". There are, he said, six developers listed as maintainers of the slab allocator, three of whom have not been seen for 20 years. (That problem is [now being addressed](<https://lwn.net/ml/all/20250407200508.121357-3-vbabka@suse.cz>)). He also complained about contributors who ""produce churn"", saying that Morton is too nice to such people. Liam Howlett said that he will normally not accept cleanup patches except as part of a larger project. Those cleanups can often break subtle things, he said. 

As the session wound down, Morton said that getting replacement patches from developers makes his life harder. He doesn't really know which reviews apply at that point, or which comments have been addressed. He would rather get small delta patches on top of the previous work, especially once that work has landed in mm-unstable. 

He closed the session by saying ""see you next year!""
