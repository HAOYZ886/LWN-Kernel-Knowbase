---
title: Page migration
url: https://lwn.net/Articles/157066/
date: "October 25, 2005"
category: "Memory management-NUMA systems; NUMA"
author: 
---

NUMA systems have, by design, memory which is local to specific nodes (groups of processors). While all memory is accessible, local memory is faster to work with than remote memory. The kernel takes NUMA behavior into account by attempting to allocate local memory for processes, and by avoiding moving processes between nodes whenever possible. Sometimes processes must be moved, however, with the result that the local-allocation optimization can quickly become a pessimization instead. What would be nice, in such situations, would be the ability to move a process's memory when the process itself is shifted to a new node. 

Memory migration patches have been circulating for some time now. The latest version is [this patch set](<http://lwn.net/Articles/156603/>) posted by Christoph Lameter. This patch deliberately does not solve the entire problem, but it does try to establish enough infrastructure that a full migration solution can be evolved eventually. 

This patch does not automatically migrate memory for processes which have been moved; instead, it leaves the migration decision to user space. There is a new system call: 

```
long migrate_pages(pid_t pid, unsigned long maxnode,
                           unsigned long *old_nodes,
                           unsigned long *new_nodes);
```

This call will attempt to move any pages belonging to the given process from `old_nodes` to `new_nodes`. There is also a new `MPOL_MF_MOVE` option to the [`set_mempolicy()`](<http://www.die.net/doc/linux/man/man2/set_mempolicy.2.html>) system call which can be used to the same effect. Either way, user space can request that a given process vacate a set of nodes. This operation can be performed in response to an explicit move of the process itself (which might be done by a system scheduling daemon, for example), or in response to other events, such as the impending shutdown and removal of a node. 

The implementation is simple for now: the code iterates over the process's memory and attempts to force each page needing migration to be swapped. When the process faults the page back in, it should then be allocated on the process's current node. The force-out process actually takes a few passes over the list; initially it passes over locked pages and just concerns itself with pages which are easy to evict. In later passes, it will wait for locked pages and do the hard work of getting the final pages out of memory. 

Migrating pages by way of the swap device is not the most efficient way of moving them across a NUMA system. Later work on the patch will be aimed at adding direct node-to-node migration, and other features as well. In the mean time, however, the developers would like to see the current implementation merged in time for 2.6.15. Andrew Morton has [expressed some reservations](<https://lwn.net/Articles/157070/>), however: he would like to see an explanation of how this code can be made to work with near complete reliability. There are a number of things which can prevent the migration of pages; these include pages locked in place by user space, page undergoing direct I/O, and more. Christoph [responded](<https://lwn.net/Articles/157074/>) that the patch will get there, eventually. Whether this claim is sufficiently convincing to get the migration patches into 2.6.15 remains to be seen.
