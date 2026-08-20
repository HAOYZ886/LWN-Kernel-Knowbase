---
title: "Fighting the zombie-memcg invasion"
url: https://lwn.net/Articles/932070/
date: "May 19, 2023"
category: "Control groups-Memory controller; Memory management-Control groups"
author: "By Jonathan Corbet May 19, 2023 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 19, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Memory control groups (or "memcgs") allow an administrator to manage the memory resources given to the processes running on a system. Often, though, memcgs seem to have memory-use problems of their own, and that has made them into a recurring Linux Storage, Filesystem, and Memory-Management Summit topic since at least [2019](<https://lwn.net/Articles/787614/>). The topic returned at [the 2023 event](<https://lwn.net/Articles/lsfmmbpf2023>) with a focus on the handling of shared, anonymous memory. The quirks associated with this memory type, it seems, can subject systems to an unpleasant sort of zombie invasion; a session in the memory-management track led by T.J. Mercier, Yosry Ahmed, and Chris Li discussed possible solutions. 

[![\[T.J. Mercier\]](https://static.lwn.net/images/conf/2023/lsfmm/TJMercier-sm.png)](<https://lwn.net/Articles/932074/>) Mercier started the session by describing how the zombie problem comes about. It all starts when a process running within a memcg allocates some anonymous memory; that memory is charged to the group and all is well. The process then allocates some shared memory, which is also duly charged. If that memory is subsequently shared with processes in a different memcg, though, things start to become a little strange; only the original owner will be charged for all that memory. The other groups can use it for free. If all of the processes in that first group go away, the memcg itself would normally be deleted, but that can't happen; it is still responsible for that shared memory, even though all of the users of that memory are outside of the group. 

That memcg has become a zombie, destined to haunt the system for, potentially, a long time. In some settings, thousands of them can accumulate, creating a true zombie horde that consumes a significant amount of kernel ~~brains~~ memory. It also slows down operations, including memory reclaim, that iterate over the memcgs in the system. This is, in other words, a problem worth fixing. 

Mercier started by going through some "non-fixes" that have been shown not to work. Forcing manual reclaim with the `memory.reclaim` memcg knob does not work if the memory is not actually reclaimable, which is the case with shared memory. Instead, it can push pages out to swap, making the problem even worse; there will be no way to get rid of the zombie memcg without first swapping any swapped-out pages back in. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Another non-fix is to try reparenting the charged memory to the zombie memcg's parent group. But that group does not actually own the memory, so this action just has the effect of hiding the zombie memory there. Reparenting can cause memory to end up in the root control group and, in general, complicates memory management. 

The fundamental problem, he concluded, was the fact that any given `page` structure can only have one memcg owner. That leaves no way to account for shared memory and leads to the zombie problem. A potential fix might be to move the charge for that memory to one of the other control groups using it; that would lead to better accounting overall, but finding that other control group is harder than it seems. A longer-term fix, Mercier said, would be to develop a first-class way to associate shared memory with multiple groups. Matthew Wilcox suggested charging the first group to access the shared memory once the owning group turns into a zombie, which led naturally to the next part of the talk. 

[![\[Yosry Ahmed\]](https://static.lwn.net/images/conf/2023/lsfmm/YosryAhmed-sm.png)](<https://lwn.net/Articles/932075/>) Ahmed then took over to talk about the option of re-charging a zombie memcg's shared-memory pages to one of the other users. The task, he said, involves iterating through the pages charged to the zombie group and looking at the type of each. Kernel pages will already have been reparented, he said, and do not need further attention. There are pages on the memcg's least-recently-used (LRU) list that may or may not be mapped, and page-cache pages that are not mapped. With these pages, there are a few options for dealing with them. 

For example, these pages can simply be evicted from memory entirely; this will work for page-cache pages, which will eventually be faulted back in and charged to the new user. This action is intrusive, though, and will slow access to heavily used pages; it also does not work with pinned pages. 

Alternatively, they can be re-charged to another group that has the page mapped; this should identify the right owner. It has the potential to charge a memcg for memory it used "hours ago", though, and could push the group into an out-of-memory situation. If multiple groups have the page mapped, the kernel would have to choose one somehow. 

Finally, a two-step "deferred re-charge" approach could be taken, where (as Wilcox had suggested) the page is charged to the group that accesses it next. The pages could be removed from the zombie group's balance sheet, perhaps charged to the parent until the right group is found. This approach would be complicated to implement and would add extra work to some hot paths. 

Ahmed sketched out an algorithm that might work, executed from a worker thread launched when the memcg enters the zombie state. This worker would iterate through the group's LRU list and take the appropriate action with each page. Unmapped, file-backed pages would simply be evicted, while unmapped anonymous pages would go through the deferred re-charge process. Pages that are mapped, instead, would be charged to a mapping group, either directly or using the deferred method. Pages that are swapped are a separate problem; they, too, will keep references to the zombie group. The answer there would be to walk through the swap cache and reparent the pages as needed. 

That, he said, might be an effective short-term solution to the problem. Michal Hocko responded that the kernel used to re-charge pages in this situation, but had to back away from that approach. It is heavily based on the idea that memcgs do not change over time, he said. The sharing of memory across memcgs is not a great idea in the first place; is there any way that it could be just avoided? Another attendee said that the kernel used to have re-_charging_ , but that re _parenting_ is different and could perhaps be more easily implemented. Telling users to not share memory between memcgs is not a good answer, he said; users are trying to save memory with more sharing, not less. Rather than avoid the problem, it would be better to just try solutions that can be implemented easily. 

[![\[Chris Li\]](https://static.lwn.net/images/conf/2023/lsfmm/ChrisLi-sm.png)](<https://lwn.net/Articles/932076/>) Ahmed added that there are a lot of ways to create sharing, some of which can be surprising. Writing a byte to a tmpfs file, for example, will nail down a page that will be stuck until the file is removed or truncated. 

Hocko said that maybe the kernel should just refuse to remove a memcg if pages remain charged to it, but Ahmed said that would just create more trouble for users. New memcgs will just be created until the machine fills up. It is difficult to see what the shared resources that are holding a memcg in place are, so users do not have an easy way to fix the problem. 

John Hubbard said that what was needed was a separate parent for shared resources that would track all of the groups using those resources. That was Li's cue to talk briefly (there was little time left at this point) about a possible long-term solution in the form of an approach for tracking those sharing relationships. The idea, which was "not fully hashed out", involved creating a separate shared-memory controller that would own memory that is shared between memcgs. Through a complicated mechanism, it would track that shared use; the result would be no movement of charges over time, and no zombie control groups. 

There was no time to discuss this idea. Chances are that some time will be found next year when this unkillable topic shows up at the 2024 conference.
