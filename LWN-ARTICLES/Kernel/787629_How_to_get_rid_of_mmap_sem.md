---
title: How to get rid of mmap_sem
url: https://lwn.net/Articles/787629/
date: "May 8, 2019"
category: "Maple trees; Memory management-mmap sem"
author: "By Jonathan Corbet May 8, 2019 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 8, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The `mmap_sem` lock used in the memory-management subsystem has been a known scalability problem for years, but it has proved difficult to remove. No less than three sessions were held during the memory-management track of the 2019 Linux Storage, Filesystem, and Memory-Management Summit to talk about `mmap_sem` and how it might be eliminated. Many ideas were discussed, but the shape of the solution remains vague at best. 

#### Maple trees

The first session, run by Laurent Dufour and Matthew Wilcox, discussed a possible solution: replacing the red-black tree currently used to track virtual memory areas (VMAs) with a new data structure called a "maple tree". 

VMAs represent the regions of memory that make up a process's address space. They are kept in an [augmented red-black tree](<https://lwn.net/Articles/388118/>) that exists to answer two kinds of queries: quickly finding the VMA associated with a given address, or finding a gap in the address space [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2019/lsfmm/WillyLaurent-sm.jpg)](<https://lwn.net/Articles/787632/>) that is large enough to hold a new VMA. There is also a separate linked list that contains the VMAs in an address space, making it possible to walk through the entire space. Either way, protection of those data structures is done with `mmap_sem`. Wilcox noted that doubly linked lists have come under a lot of criticism in recent years; they are "the devil's structure" with poor performance characteristics. The current structure for VMAs, he said, is a "quintuply linked list" with even worse performance. 

Naturally (since he is the creator of the XArray structure), Wilcox wanted to replace everything with an XArray. It is, he said, "absolutely superb" for the page cache, with all the properties one might want for looking up pages with a 2n-byte alignment. But the XArray lacks good range support, which is what is needed here; there is no way to search for gaps in the address space. It is, thus, not a good structure for this task. 

So, instead, he suggests using maple trees, a data structure he has been working on with Liam Howlett. It is a form of B-tree that has been optimized for storing ranges that lack significant gaps, which is just what the VMA list is. VMAs are mostly adjacent, he said, with a relatively small number of gaps between them. Hugh Dickins predicted that somebody was sure to post a patch in the near future that spread out VMAs as a hardening measure; Wilcox responded that the structure could be tweaked accordingly if that ever happens. 

One of the key features of the new structure is lockless access to VMAs, a claim that brought an immediate protest from Mel Gorman, who worried that it could never be safe. One of the core aspects of the VMA, he said, is that it is actually two data structures in disguise. One of them is the VMA itself; the other is the `address_space` structure describing what that range of memory actually maps to. The two structures have their own life cycles and locking schemes, and lockless access will be hazardous at best. It might be necessary to add some sort of reference counter to the address space to keep it from going away while the VMA is being worked on. 

That, Wilcox said, is why this discussion was happening; it would allow the developers to get a sense for what the end state would look like. Meanwhile, while the maple tree conversion is happening, code will continue [![\[Laurent Dufour\]](https://static.lwn.net/images/conf/2019/lsfmm/WillyLaurent-sm2.jpg)](<https://lwn.net/Articles/787632/>) to use `mmap_sem` to access the data structure. There are two phases to this work: replacing the data structure with something more efficient, and separating out the locking. 

Andrea Arcangeli said that it might be better to do things in the opposite order, and find ways to fix the locking first. For example, he said there should be no need to use `mmap_sem` with `VM_GROWSDOWN` VMAs (those representing stack areas, normally). It can also be avoided for calls to `get_user_pages_fast()`. The complexity of the code may be increased by removing the locking first, but he said that it was pointless to try to change the two things together. It is better to change the locking first; otherwise you can't really even benchmark the results of the other changes. 

Michal Hocko was unsure about changing the locking first, but he liked the idea of getting rid of the doubly linked list and doing lockless searches. This work could also help, he said, with range locking, which is a commonly suggested way of reducing contention on `mmap_sem`. Rather than using `mmap_sem` to lock the entire range, taking a range lock on just the part of the address space being worked on would allow more to happen in parallel. He asked: how much work is needed to make this happen? 

Wilcox replied that the patch, so far, is about 2,000 lines of code, but it's addressing more than just the VMA problem. There are three users of radix trees for ranges; all are horrible, he said, and should be replaced. There are other places using red-black trees that could be improved as well. In the end, this work will give him the opportunity to delete a lot of code. The maple tree structure will be there anyway, regardless of whether it is used for VMAs. Hocko said that was a good argument for using it with VMAs too. 

At the end of the session, the next steps were laid out. Some sort of per-VMA locking will be set up, probably based on a range lock. Hocko suggested that the range locking should happen first, since adding range locks is a huge step. There are many places where `mmap_sem` is abused to protect other data structures; it will take a long time just to figure out where they all are. Gorman suggested that all `mmap_sem` users should be changed to acquire a lock on the entire address-space range; after that, each use can be evaluated to see if it can be narrowed. That will allow some sort of incremental progress. This task is, he said, "the big kernel lock all over again". 

#### Range locks

Wilcox led a session on the following day dedicated to range locks in particular. Range locking is, he said, "a very seductive idea". There is a highly contended lock getting in the way, so it is natural to want to split it up. After all, the kernel doesn't use one big lock for access to inodes; instead, each inode has its own lock. Something similar has obvious appeal for the memory-management subsystem. 

Davidlohr Bueso has done [a range-lock implementation](<https://lwn.net/Articles/724502/>) for the kernel that stores ranges in a [red-black tree](<https://lwn.net/Articles/184495/>). But, Wilcox said, he is not a big fan of red-black trees; he has a replacement to propose (presumably the maple tree) that should perform better, but it is not working yet. That said, he is also not a fan of range locks, which he finds inelegant. They use one complicated data structure to protect another; in this case, one red-black tree (the range locks) is being used to protect another red-black tree (containing the VMAs). It would be better, he said, to build the locking into the tree itself. 

There is some data to back this up. Dave Chinner tried to use range locks to improve XFS, Wilcox said, and ended up with a significant slowdown. Mel Gorman protested, though, that Chinner's experiment used an XFS-specific range lock that managed extents; it is not an equivalent situation, he said. 

Wilcox then admitted that he did not have a whole lot else to say. Gorman said that it would be great to have a lock in each VMA, but that is a lot harder to do for holes in the address space. No object exists there yet, so there is nothing to put a lock into. But evidently the DAX subsystem has a solution to that problem now, inserting locked entries into holes in the radix tree. Wilcox is planning something similar for VMAs, but there is no code yet so it is too early to talk about now. 

At this point, this relatively short session came to a close. 

#### mmap_sem again

At the very end of the conference, Dufour ran one more session with the desultory title of "`mmap_sem` again". It was ostensibly about the [speculative page faults](<https://lwn.net/Articles/730531/>) work, though it did not focus much on that work specifically. 

One purpose of `mmap_sem` is to protect VMAs against concurrent changes. That could perhaps be replaced in a twofold manner: a lock to protect the list of VMAs (perhaps using read-copy-update), and a lock embedded in each individual VMA. The latter lock would have to be a sleeping lock, since it must be taken while handling page faults. That leads immediately to the first challenge: how does one go from the non-sleeping lock protecting the list to the sleeping lock for the VMA? Acquiring the first lock will normally invoke a context where sleeping is no longer an option. 

There was some talk of using reference counts for this purpose; Wilcox described it as "open-coding a semaphore". Gorman said there would have to be a wait queue for anybody who needs to wait for the reference count to go to zero. Starvation could become an issue in some settings. He also advised against using RCU, which is only useful when getting a copy of an object is sufficient; that is never true for VMAs. There would need to be a convincing explanation of how all this is actually better than `mmap_sem`, he said. 

Jérôme Glisse suggested adding counts to the VMA so that all faults could be handled speculatively. But Gorman argued against building on the speculative page-fault code, which has yet to produce any performance gain in any of his tests. Glisse said that he was only thinking about taking parts of it to check for concurrent changes to a VMA while other work is going on. He would like to avoid range locking, he said; instead, the page-table locks can function as a sort of natural range lock. Hocko disagreed with the idea that range locks should be avoided. An address space is a collection of ranges that the memory-management subsystem operates on, so a range lock is a natural solution to the problem. 

Wilcox turned the discussion back to `mmap_sem`, noting that it is highly contended and wondering how it could be split up. Part of the problem, he said, is that `mmap_sem` "covers many sins" beyond the protection of VMAs. Once that is dealt with, though, everybody agrees that work needs to be separated by range; that is not the point of contention. Instead, the dispute centers around whether Bueso's range locks, in particular, should be used. 

For a next step, Glisse brought back the idea of replacing `mmap_sem` with a wrapper that would lock the entire range; Wilcox responded that this approach was "crazy". His own next step is moving unrelated data out from under `mmap_sem` to simplify the problem. Meanwhile, he is continuing to work on the maple tree concept as an independent effort. He also said that working on speculative page faults is valuable, even if it yields no immediate performance benefits; it helps the developers discover the perils that come with splitting up `mmap_sem` in general. Dufour agreed that he had already learned far more than he had ever wanted while working on that code; he is not sure about the future of speculative page faults but hopes that it can help to get to a more scalable memory-management subsystem. 

Dave Hansen complained that `mmap_sem` is a heavyweight lock, even when just acquired for reading. It bounces its reference count around the system, creating a lot of cache misses. Adding a reference count to the VMA structure would, he said, just move the problem. Dufour said that the current practice of merging VMAs whenever possible might be making things worse by increasing contention on the locks; perhaps merging should not be done so aggressively. 

The session was coming to an end, and the participants were clearly tired after three days of this kind of discussion. Hocko said that the group should at least come up with an action item. The goal is to replace `mmap_sem`; if developers don't like range locks, then what would they suggest using instead? It needs to be something that, like range locks, can turn the `mmap_sem` replacement into an incremental problem so that the transition is manageable. The focus should be on the new API, he said; once that is sane, developers can work on the implementation. 

The problem, Wilcox countered, is that the range-lock API doesn't work for this problem. Code often does not know the size of the range it needs when it takes the lock, so it ends up locking the entire region, then downgrading to something smaller. But locking everything for every fault sounds a lot like `mmap_sem`, and downgrading will be an extra expense on top of that. Hocko asked what the alternative was, if there is not an API to start with, but no definitive answer was at hand. As the session concluded, Gorman said that one action item would be to get Bueso to repost the work he as done so far; perhaps there are some lessons to be learned from it.
