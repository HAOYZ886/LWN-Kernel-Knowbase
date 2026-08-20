---
title: Improving the OOM killer
url: https://lwn.net/Articles/684945/
date: "April 27, 2016"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jonathan Corbet April 27, 2016 LSFMM 2016"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

When the system becomes so short of memory that nothing can make forward progress, the only possible way to salvage anything may be to kill processes until some memory becomes available. That is where the dreaded out-of-memory (OOM) killer comes into play. For as long as the OOM killer has existed, developers have been trying to improve its operation. The latest to work in this area is Michal Hocko, who led a session during the memory-management track of the 2016 Linux Storage, Filesystem, and Memory-Management Summit to talk about what he has been doing. 

One of Michal's goals is to make the process of detecting OOM situations more reliable and deterministic. How things have been done in practice so far, he said, is to try to reclaim memory until nothing more can be found for several iterations in a row, then invoke the OOM killer. The problem is that there have always been [![\[Michal Hocko\]](https://static.lwn.net/images/conf/2016/lsfmm/MichalHocko2-sm.jpg)](<https://lwn.net/Articles/684949/>) bugs in this code. The OOM killer is only summoned for order-0 (single-page) requests and, worse, a single free page resets the scan counter. That means that, with a tiny trickle of pages becoming free, the kernel can loop forever without ever starting the OOM killer. 

Michal's [work](<https://lwn.net/Articles/668126/>) in this area involves getting feedback from the reclaim and compaction code, and invoking the OOM killer if the situation doesn't improve over time. In the future, he would like to make the code more conservative, and to detect when the system is thrashing. In thrashing situations, the OOM killer could be started even if the system is not strictly out of memory. Christoph Lameter complained that starting the OOM killer "wrecks the system" by killing off useful processes, but Michal responded that, in such situations, the system is already lost, so it makes sense to try to recover it partially. Then, if nothing else, an administrator can get in and try to figure out what the problem is. The situation as it exists now is fragile, he said, and is worth changing. The developers in the room seemed to agree with that sentiment, and it was decided that this work should be merged. 

Michal's other area of work is OOM-killing reliability — making sure that something useful happens after the OOM killer is started. Some developers have been trying to add timeouts to the OOM-killing code, meaning that, if killing one process does not yield free memory within a bounded time, the OOM killer would move on to a new victim. Michal has been pushing back on those, in the opinion that other means should be used if possible. His alternative is the [OOM reaper](<https://lwn.net/Articles/668126/#reaper>), which deprives a victim process of its memory resources even before that process can exit. That allows the memory to be freed even if the victim process is blocked on some lock and unable to exit. This code was merged for the 4.6 release. 

While nobody objected to that work, some developers still felt that there is a place for timeouts in the OOM killer code. There are situations, for example, in which the OOM reaper will be unable to free a process's memory. Should things get wedged, the feeling seemed to be, it's better to try killing another process than to lose the system altogether. Michal said that, if somebody wants to work on adding timeouts, it would be acceptable to him as long as the code was deterministic. Timeouts are, after all, orthogonal to the rest of what he is working on. Andrea Arcangeli warned against attempts to make the OOM killer perfect, since it is unlikely to ever get there. 

As the session came to a close, Hugh Dickins raised another problem: what to do if all of the system's memory is tied up in the tmpfs filesystem (which has no backing store and only stores files in memory). Killing processes will not, in general, cause that memory to be freed, and there is, he said, no way to randomly truncate files to free their pages. There is an experiment in Google, he said, to try to truncate large tmpfs files when the system runs out of memory. The immediate reaction in the room, though, was that any such approach is dangerous at best, so this patch may not ever make it out into the wider world.
