---
title: "Time to thrash the 2.6 VM?"
url: https://lwn.net/Articles/74149/
date: "March 3, 2004"
category: Memory management
author: 
---

Those who have been watching kernel development for a little while will remember [the fun](<http://lwn.net/2001/0927/kernel.php3>) that came with the 2.4.10 release, when Linus replaced the virtual memory subsystem with a new implementation by Andrea Arcangeli. The 2.4 kernel did end up with a stable VM some releases thereafter, but many developers were upset that such a major change would be merged that far into a stable series. Especially since many of those developers were not convinced that the previous VM was not fixable. 

The 2.4 changes are long past, but the memories are fresh enough that when Andrea put forward [a set of VM changes](<https://lwn.net/Articles/73367/>) which, while they are for 2.4, are said to be applicable to 2.6 as well, people took notice. Andrea's goals this time are little more focused; he is concerned with the performance of systems with at least 32GB of installed memory and hundreds of processes with shared mappings of large files. This, of course, is the sort of description that might fit a high-end database server. 

Andrea has found three problems which make those massive servers fail to function well. The first has to do with how 2.4 performs swapout; it works by scanning each process's virtual address space, and unmapping pages that it would like to make free. When a page's mapping count reaches zero, it gets kicked out of main memory. The problem is that this algorithm performs poorly in situations where many processes have the same, large file mapped. The VM will start by unmapping the entire file for the first process, then another, and so on. Only when it has passed through all of the processes mapping the file can it actually move pages out of main memory. Meanwhile, all of those processes are incurring minor page faults and remapping the pages. With enough memory and processes, the VM subsystem is almost never able to actually free anything. 

This is the problem that the reverse-mapping VM (rmap) was added to 2.5 to solve. By working directly with physical pages and following pointers to the page tables which map them, the VM subsystem can quickly free pages for other use. Andrea is critical of rmap, however; with his scenario of 32GB of memory and hundreds of processes, the rmap infrastructure grows to a point where the system collapses. Instead, for his patches, he has implemented a variant of the [object-based reverse mapping](<https://lwn.net/Articles/23732/>) scheme. Object-based reverse mapping works by following the links from the object (a shared file, say) which backs up the shared memory; in this way it is able to dispense with the rmap structures in many situations. There are some concerns about pathological performance issues with the object-based approach, but those problems do not seem to arise in real-world use. 

The second problem is a simple bug in the swapout code. When shared memory is unmapped and set up for swap, the actual I/O to write it out to the swap file is not started right away. By the time the system gets around to actually performing I/O, there is a huge pile of pages waiting to be shoved out, and an I/O storm results. Even then, the way the kernel tracks this memory means that it takes a long time to notice that it is free even after it has been written to swap. This problem is fixed by taking frequent breaks to actually shove dirty memory out to disk. 

Andrea's final problem came about when he tried to copy a large file while all those database processes were running. It turns out that the system was swapping out the shared database memory (which was dirty and in use) rather than the data from the file just copied (which is clean). Tweaking the memory freeing code to make it prefer clean cache pages over dirty pages straightened this problem out, at the cost of a certain amount of unfairness. 

With these patches, Andrea claims, the 2.4 kernel can run heavy loads on large systems which will immediately lock up a 2.6 system. So he is going to start looking toward 2.6, with an eye toward beefing it up for this sort of load. Andrew Morton has [indicated](<https://lwn.net/Articles/74155/>) that he might accept some of this work - but not yet: 

We need to understand that right now, 2.6.x is 2.7-pre. Once 2.7 forks off we are more at liberty to merge nasty highmem hacks which will die when 2.6 is end-of-lined. 

I plan to merge the 4g split immediately after 2.7 forks. I wouldn't be averse to objrmap for file-backed mappings either - I agree that the search problems which were demonstrated are unlikely to bite in real life. 

The "4g split" is Ingo Molnar's [4GB user-space patch](<https://lwn.net/Articles/39925/>) which makes more low memory available to the kernel, but at a performance cost. Before Andrew merges any other patches, however, he wants to see a convincing demonstration of why the current VM patches are not enough for large loads. The 2.6 "stable" kernel may well see some significant virtual memory work, but, with luck, it will not be subjected to a 2.4.10-like abrupt switch.
