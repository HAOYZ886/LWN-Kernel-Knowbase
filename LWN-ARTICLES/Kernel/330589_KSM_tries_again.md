---
title: KSM tries again
url: https://lwn.net/Articles/330589/
date: "April 28, 2009"
category: "Memory management-Kernel samepage merging"
author: "By Jonathan Corbet April 28, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 28, 2009

Back in November, LWN [examined the KSM (kernel shared memory) patch](<http://lwn.net/Articles/306704/>). KSM was written to address a problem encountered on systems running virtualized guests: such systems can have large numbers of pages holding identical data, but the kernel has no way to let guests share those pages. The KSM code scans through memory to find pairs of pages holding identical data; when such pairs are found, they are merged into a single page mapped into both locations. The pages are marked copy-on-write, so the kernel will automatically separate them again should one process modify its data. 

There were some concerns about the intended purpose of this patch, but it was soon made clear that KSM can help the system to save significant amounts of memory. But KSM was not merged as a result of two other problems. One of them, discussed mostly behind closed doors, appears to be concerns about the use of SHA1 hashes to compare pages. If an attacker could create hash collisions, he might just be able to inject his own data (or code) into processes belonging to other users and/or virtual machines. The other problem had to do with a different kind of attacker: [VMWare holds a patent](<http://lwn.net/Articles/309155/>) to an algorithm which looks quite similar to the method used by the early KSM patches. There is evidence that this patent could be overturned on prior art, but that is still a battle that nobody wants to fight. 

KSM disappeared from view for a while after those issues came to light, but, more recently, new versions of the KSM patches have been [posted](<http://lwn.net/Articles/329123/>) for review. A quick look at the code makes it clear that both of these concerns have been addressed \- and, in fact, that the KSM developers were able to kill both birds with the same stone. It's all a matter of doing away with the hashing of pages. 

[Patent 6,789,156](<http://www.google.com/patents?vid=USPAT6789156>) is not exactly light reading; it has a full 74 claims. Most of the independent claims have one thing in common, though: they include the calculation of a hash value to find identical pages in the system. If the KSM code were to avoid hashing pages, those claims of the patent would clearly not read against it. And, as described above, the use of hashing also created some security worries. So it must have seemed clear to the KSM developers (and any lawyers they may have talked to) that the hash needed to go. 

The current KSM patches have replaced the hash table with two separate [red-black trees](<http://lwn.net/Articles/184495/>). Pages tracked by KSM are initially stored in the "unstable tree"; the term "unstable" means that KSM considers their contents to be volatile. Placement in the tree is determined by a simple `memcmp()` of the page's contents; essentially, the page is treated as containing a huge number and sorted accordingly. The unstable tree is suitable for finding pages with duplicate contents; a relatively quick traversal of the tree will turn up the only candidates. 

It's worth noting that KSM does not place every page it scans in the unstable tree. If the contents of a page change over the course of one memory scanning cycle, the page will not really be a good candidate for sharing anyway. So pages which are seen to change are not represented in the unstable tree. The unstable tree is also dumped and rebuilt from the beginning after each scan cycle. That deals with the problem of pages which, as a result of modifications, find themselves in the wrong location in the tree. The nature of red-black trees means that search and insertion operations are almost the same thing, so there is little real cost to rebuilding the unstable tree from the beginning every time. 

The other pages which are not found in the unstable tree are those which have actually been merged with duplicates. Since shared pages are marked read-only, KSM knows that their contents cannot change. Those pages are put into a separate "stable tree." The stable tree is also a red-black tree, but, since pages cannot become misplaced there, it need not be rebuilt regularly. Once a page goes into the stable tree, it stays there until all users have either modified or unmapped it. 

The resulting system clearly works. Dropping the hash may impose a cost in the form of slightly higher CPU and memory use; there have been no benchmarks posted which would show the difference. But there is no cost on systems which do not use KSM at all, and, in any case, avoiding the potential problems associated with using hash tables to identify pages with identical contents will be worth the cost. At this point, comments on the KSM code are mostly concerned with relatively small details. It could well be that this code will be ready for inclusion into the 2.6.31 kernel. 

(Postscript: above, your editor noted that "most" of the independent claims in the VMWare patent required the use of a hashing mechanism. There are, in fact, a few claims which lack that requirement, but they replace it with one or two others. Some claims cover the use of copy-on-write pages, but they all explicitly say that this technique is used on pages with a "relatively high probability of impending modification." But there is little point in sharing such pages at all; KSM, by leaving them out of the unstable tree, ignores those pages entirely. The remaining claims describe partitioning memory into "classes," which is not done in KSM.)
