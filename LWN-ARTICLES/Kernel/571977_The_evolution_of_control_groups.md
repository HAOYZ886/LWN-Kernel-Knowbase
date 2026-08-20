---
title: The evolution of control groups
url: https://lwn.net/Articles/571977/
date: "October 29, 2013"
category: Control groups
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

The control group (cgroup) subsystem is currently under intensive development; some of that work will lead, eventually, to ABI changes that are visible from user space. Given the amount of controversy around this subsystem, it was not surprising to see control groups show up on the 2013 Kernel Summit agenda. Tejun Heo led a session to describe the consensus that had been reached on the cgroup interface, only to find that there are still a few loose ends to be tied down. 

Tejun started by reminding the group that the multiple hierarchy feature of cgroups, whereby processes can be placed in multiple, entirely different hierarchies, is going away. The unified hierarchy work is not entirely [![\[Tejun Heo\]](https://static.lwn.net/images/conf/2013/lce-ks/TejunHeo-sm.jpg)](<https://lwn.net/Articles/572012/>) usable yet, though, because it requires that all controllers be enabled for the full hierarchy. Some controllers still are not hierarchical at all; they are being fixed over time. The behavior of controllers is being made more uniform as well. 

One big change that has been decided upon recently is to make cgroup controllers work on a per-process basis; currently they apply per-thread instead. Among other things, that means that threads belonging to the same process can be placed in different control groups, leading to various headaches. Of all the controllers only the CPU controller has any business working with individual threads. For that case, some sort of special interface will be introduced that will, among other things, allow processes to set CPU policies for their own threads. 

That interface, evidently, might be implemented with yet another special-purpose virtual filesystem. There was some concern about how the cgroup subsystem may be adding features that, essentially, constitute new system calls without review; there were also concerns about how the filesystem-based interface suffers from race conditions. Peter Zijlstra worried about how the new per-thread interface might look, saying that there were a lot of vague details that still need to be worked out. Linus wondered if it was really true that only the CPU controller needs to look at individual threads; some server users, he said have wanted per-thread control for other resources as well. 

Linus also warned that it might not be possible to remove the old cgroup interface for at least ten years; as long as somebody is using it, it will need to be supported. Tejun seemed unworried about preserving the old interface for as long as it is needed. Part of Tejun's equanimity may come from a feeling that it will not actually be necessary to keep the old interface for that long; he said that even Google, which has complained about the unified hierarchy plans in the past, has admitted that it can probably make that move. So he doesn't see people needing the old interface for a long time. 

In general, he said, the biggest use for multiple hierarchies has been to work around problems in non-hierarchical controllers; once those problems are fixed, there will be less need for that feature. But he still agrees that it will need to be maintained for some years, even though removal of multiple hierarchy support would simplify things a lot. Linus pointed out that, even if nobody is using multiple hierarchies currently, new kernels will still need to work on old distributions for a long time. Current users can be fixed, he said, but Fedora 16 cannot. 

Hugh Dickins worried that, if the old interface is maintained, new users may emerge in the coming years. Should some sort of warning be added to tell those users to shift to the new ABI? James Bottomley said, to general agreement, that deprecation warnings just don't work; distributions just patch them out to avoid worrying their users. Tejun noted that new features will only be supported in the new ABI; that, hopefully, will provide sufficient incentive to use it. Hugh asked what would happen if somebody submitted a patch extending the old ABI; Tejun said that the bar for acceptance would be quite high in that case. 

From the discussion, it was clear that numerous details are still in need of being worked out. Paul Turner said that there is a desire for a notification interface for cgroup hierarchy changes. That, he said, would allow a top-level controller to watch and, perhaps, intervene; he doesn't like that idea, since Google wants to be able to delegate subtrees to other processes. In general, there seems to be a lack of clarity about who will be in charge of the cgroup hierarchy as a whole; the systemd project has plans in that area, but that creates difficulties when, for example, a distribution is run from within a container. Evidently some sort of accord is in the works there, but there are other interesting questions, such as what happens when the new and old interfaces are used at the same time. 

All told, there is a fair amount to be decided still. Meanwhile, Tejun said, the next concrete step is to fix the locking, which is currently too strongly tied to the internal locking of the virtual filesystem layer. After that is done, it should be possible to post a prototype showing how the new scheme will work. That posting may happen by the end of the year. 

[Next: [Linux-next and -stable](<https://lwn.net/Articles/571980/>)].
