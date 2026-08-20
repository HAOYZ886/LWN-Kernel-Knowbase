---
title: "Preparing for large-sector drives"
url: https://lwn.net/Articles/582862/
date: "January 29, 2014"
category: "Block layer-Large physical sectors; Memory management"
author: "By Jonathan Corbet January 29, 2014"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 29, 2014

Back in the distant past (2010), kernel developers were [working on supporting drives with 4KB physical sectors](<https://lwn.net/Articles/377895/>) in Linux. That work is long since done, and 4KB-sector drives work seamlessly. Now, though, the demands on the hard drive industry are pushing manufacturers toward the use of sectors larger than 4KB. A recent discussion ahead of the upcoming (late March) Linux Storage, Filesystem and Memory Management Summit suggests that getting Linux to work on such devices may be a rather larger challenge requiring fundamental kernel changes — unless it isn't. 

Ric Wheeler [started the discussion](<https://lwn.net/Articles/582879/>) by proposing that large-sector drives could be a topic of discussion at the Summit. The initial question — when such drives might actually become reality — did not get a definitive answer; drive manufacturers, it seems, are not ready to go public with their plans. Clarity increased when Ted Ts'o revealed [a bit of information](<https://lwn.net/Articles/582881/>) that he was able to share on the topic: 

In the opinion of at least one drive vendor, the pressure for 64k sectors will start increasing (roughly paraphrasing that vendor's engineer, "it's a matter of physics"), and it might not be surprising that in 2 or 3 years, we might start seeing drives with 64k sectors. 

Larger sectors would clearly bring some inconvenience to kernel developers, but, since they can help drive manufacturers offer more capacity at lower cost, they seem almost certain to show up at some point. 

#### Do (almost) nothing

One possible response, espoused by James Bottomley, is to do very little in anticipation of these drives. He pointed out that much of the work done to support 4KB-sector drives was not strictly necessary; the drive manufacturers said that 512-byte transfers would not work on such drives, but the reality has turned out to be different. Not all operating systems were able to adapt to the 4KB size, so drives have read-modify-write (RMW) logic built into their firmware to handle smaller transfers properly. So Linux would have worked anyway, albeit with some performance impact. 

James's point is that the same story is likely to play out with larger sector sizes; even if manufacturers swear that only full-sector transfers will be supported, those drives will still, in the end, have to work with popular operating systems. To do that, they will have to support smaller transfers with RMW. So it comes down to what's needed to perform adequately on those drives. Large transfers will naturally include a number of full-sector chunks, so they will mostly work already; the only partial-sector transfers would be the pieces at either end. Some minor tweaks to align those transfers to the hardware sector boundary would improve the situation, and a bit of higher-level logic could cause most transfers to be sized to match the underlying sector size. So, James [said](<https://lwn.net/Articles/582886/>): 

I'm asking what can we do with what we currently have? Increasing the transfer size is a way of mitigating the problem with no FS support whatever. Adding alignment to the FS layout algorithm is another. When you've done both of those, I think you're already at the 99% aligned case, which is "do we need to bother any more" territory for me. 

But Martin Petersen, arguably the developer most on top of what manufacturers are actually doing with their drives, [claimed](<https://lwn.net/Articles/582887/>) that, while consumer-level drives all support small-sector emulation with RMW, enterprise-grade drives often do not. If the same holds true for larger-sector drives, the 99% solution may not be good enough and more will need to be done. 

#### Larger blocks in the kernel

There are many ways in which large sector support could be implemented in the kernel. One possibility, [mentioned](<https://lwn.net/Articles/582900/>) by Chris Mason, would be to create a mapping layer in the device mapper that would hide the larger sector sizes from the rest of the kernel. This option just moves the RMW work into a low-level kernel layer, though, and does nothing to address the performance issues associated with that extra work. 

Avoiding the RMW overhead requires that filesystems know about the larger sector size and use a block size that matches. Most filesystems are nearly ready to do that now; they are generally written with the idea that one filesystem's block size may differ from another. The challenges are, thus, not really at the filesystem level; where things get interesting is with the memory management (MM) subsystem. 

The MM code deals with memory in units of pages. On most (but not all) architectures supported by Linux, a page is 4KB of memory. The MM code charged with managing the page cache (which occupies a substantial portion of a system's RAM) assumes that individual pages can easily be moved to and from the filesystems that provide their backing store. So a page fault may just bring in a single 4KB page, without regard for the fact that said page may be embedded within a larger sector on the storage device. If the 4KB page cannot be read independently, the filesystem code must read the whole sector, then copy the desired page into its destination in the page cache. Similarly, the MM code will write pages back to persistent store with no understanding of the other pages that may share the same hardware sector; that could force the filesystem code to reassemble sectors and create surprising results by writing out pages that were not, yet, meant to be written. 

Avoiding these problems almost certainly means teaching the MM code to manage pages in larger chunks. There have been some attempts to do so over the years; consider, for example, Christoph Lameter's [large block patch set](<https://lwn.net/Articles/232757/>) that was covered here back in 2007. This patch enabled variable-sized chunks in the page cache, with anything larger than the native page size being stored in compound pages. And that is where this patch ran into trouble. 

Compound pages are created by grouping together a suitable number of physically contiguous pages. These "higher-order" pages have always been risky for any kernel subsystem to rely on; the normal operation of the system tends to fragment memory over time, making such pages hard to find. Any code that allocates higher-order pages must be prepared for those allocations to fail; reducing the reliability of the page cache in this way was not seen as desirable. So this patch set never was seriously considered for merging. 

Nick Piggin's [fsblock work](<https://lwn.net/Articles/322668/>), also started in 2007, had a different goal: the elimination of the "buffer head" structure. It also enabled the use of larger blocks when passing requests to filesystems, but at a significant cost: all filesystems would have had to be modified to use an entirely different API. Fsblock also needed higher-order pages, and the patch set was, in general, large and intimidating. So it didn't get very far, even before Nick disappeared from the development community. 

One might argue that these approaches should be revisited now. The introduction of transparent huge pages, [memory compaction](<https://lwn.net/Articles/368869/>), and more, along with larger memory sizes in general, has made higher-order allocations much more reliable than they once were. But, as Mel Gorman [explained](<https://lwn.net/Articles/582901/>), relying on higher-order allocations for critical parts of the kernel is still problematic. If the system is entirely out of memory, it can push some pages out to disk or, if really desperate, start killing processes; that work is guaranteed to make a number of single pages available. But there is nothing the kernel can do to guarantee that it can free up a higher-order page. Any kernel functionality that depends on obtaining such pages could be put out of service indefinitely by the wrong workload. 

#### Avoiding higher-order allocations

Most Linux users, if asked, would not place "page cache plagued by out-of-memory errors" near the top of their list of desired kernel features, even if it comes with support for large-sector drives. So it would seem that any scheme based on being able to allocate physically contiguous chunks of memory larger than the base allocation size used by the MM code is not going to get very far. The alternatives, though, are not without their difficulties. 

One possibility would be to move to the use of virtually contiguous pages in the page cache. These large pages would still be composed of a multitude of 4KB pages, but those pages could be spread out in memory; page-table entries would then be used to make them look contiguous to the rest of the kernel. This approach has special challenges on 32-bit systems, where there is little address space available for this kind of mapping, but 64-bit systems would not have that problem. All systems, though, would have the problem that these virtual pages are still groups of small pages behind the mapping. So there would still be a fair amount of overhead involved in setting up the page tables, creating scatter/gather lists for I/O operations, and more. The consensus seems to be that the approach could be workable, but that the extra costs would reduce any performance benefits considerably. 

Another possibility is to increase the size of the base unit of memory allocation in the MM layer. In the early days, when a well-provisioned Linux system had 4MB of memory, the page size was 4KB. Now that memory sizes have grown by three orders of magnitude — or more — the page size is still 4KB. So Linux systems are managing far more pages than they used to, with a corresponding increase in overhead. Memory sizes continue to increase, so this overhead will increase too. And, as Ted [pointed out](<https://lwn.net/Articles/582904/>) in a different discussion late last year, persistent memory technologies on the horizon have the potential to expand memory sizes even more. 

So there are good reasons to increase the base page size in Linux even in the absence of large-sector drives. As Mel [put it](<https://lwn.net/Articles/582901/>), ""It would get more than just the storage gains though. Some of the scalability problems that deal with massive amount of struct pages may magically go away if the base unit of allocation and management changes."" There is only one tiny little problem with this solution: implementing it would be a huge and painful exercise. There have been [attempts](<https://lwn.net/Articles/23785/>) to implement "page clustering" in the kernel in the past, but none have gotten close to being ready to merge. Linus has also been somewhat hostile to the concept of increasing the base page size in the past, fearing the memory waste caused by internal fragmentation. 

#### A number of unpleasant options

In the end, Mel described the available options in this way: 

So far on the table is 

  1. major filesystem overhaul 
  2. major vm overhaul 
  3. use compound pages as they are today and hope it does not go completely to hell, reboot when it does 

With that set of alternatives to choose from, it is not surprising that none have, thus far, developed an enthusiastic following. It seems likely that all of this could lead to a most interesting discussion at the Summit in March. Even if large-sector drives could be supported without taking any of the above options, chances are that, sooner or later, the "major VM overhaul" option is going to require serious consideration. It may mostly be a matter when somebody feels the pain badly enough to be willing to try to push through a solution.
