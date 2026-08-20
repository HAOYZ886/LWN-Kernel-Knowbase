---
title: "How many page flags do we really have?"
url: https://lwn.net/Articles/335768/
date: "June 3, 2009"
category: "Memory management-struct page"
author: "By Jonathan Corbet June 3, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 3, 2009

The recently-discussed [kernel memory sanitization patch](<http://lwn.net/Articles/334747/>) was criticized on a number of points, one of which was its use of a dedicated page flag. Andi Kleen's [HWPOISON patch](<http://lwn.net/Articles/334928/>) (enabling upcoming Intel CPU features for dealing with memory errors) have run into trouble on similar grounds. The desperate shortage of page flags has been an article of faith among kernel developers for years. But, interestingly, not everybody agrees that a problem exists, and almost nobody can answer the simple question of how many flags are available in the first place. So a look at the Linux page flags issue seems in order. 

"Page flags" are simple bit flags describing the state of a page of physical memory. They are defined in `<linux/page-flags.h>`. Flags exist to mark "reserved" pages (kernel memory, I/O memory, or simply nonexistent), locked pages, those under writeback I/O, those which are part of a compound page, pages managed by the slab allocator, and more. Depending on the target architecture and kernel configuration options selected, there can be as many as 24 individual flags defined. 

These flags live in the `flags` field of `struct page`. This field is declared to be an `unsigned long`, so one might think that figuring out how much space is left for new flags would be a straightforward task. To a casual observer, it would look like, on a 32-bit system, 24 flags have been used, leaving eight available: 

> ![\[Page
flags\]](https://static.lwn.net/images/ns/kernel/page-flags.png)

In other words, the situation is starting to get tight, but it is not a crisis quite yet. 

But little is straightforward when it comes to `struct page`. One of these structures exists for every physical page in the system; on a 4GB system, there will be one million `page` structures. Given that every byte added to `struct page` is amplified a million times, it is not surprising that there is a strong motivation to avoid growing this structure at any cost. So `struct page` contains no less than three unions and is surrounded by complicated rules describing which fields are valid at which times. Changes to how this structure is accessed must be made with great care. 

Unions are not the only technique used to shoehorn as much information as possible into this small structure. Non-uniform memory access (NUMA) systems need to track information on which node each page belongs to, and which zone within the node as well. Rather than add fields to `struct page`, the NUMA hackers grabbed the free bits at the top of the `flags` field, yielding something like this: 

> ![\[Page
flags\]](https://static.lwn.net/images/ns/kernel/page-flags1.png)

So, on a 32-bit system with 24 page flags defined (a pessimistic scenario), there are eight bits available for the node and zone information, practically limiting 32-bit NUMA systems to 64 nodes, which is almost certainly adequate. But the addition of more page flags would come at the cost of supporting fewer NUMA nodes, and that would be unwelcome. 

Things get worse on systems with complicated physical memory layouts. On such systems, memory is not organized into a single, continuous range of physical addresses; instead, it is spread out with holes in the middle. Memory management on these "sparse memory" systems requires that each page have a "section" number associated with it. That section number is stored - you guessed it - in the spare bits at the top of the `flags` field. If space gets too tight, the kernel will move the node number into a separate array, slowing things down in the process. Either way, it seems clear that there is not a whole lot of spare room in the `flags` field on these systems. 

So the real answer to "how many page flags are free?" is, for all practical purposes, "zero," at least on 32-bit NUMA systems. Making room for more would require expanding `struct page`, which is a heavy cost to pay. Developers should, thus, not be surprised when proposals to use new page flags run into stiff opposition. It's only one bit, but that bit is in the middle of some of the most sought-after real estate in the entire kernel. 

In the case of Andi's HWPOISON patch, this opposition has come in the form of a number of alternative suggestions. [One](<https://lwn.net/Articles/335777/>) was to simply use the "reserved" bit, but that could lead to difficulties in parts of the code where that usage is not expected. Then it was [suggested](<https://lwn.net/Articles/335778/>) that the combination of the "reserved" and "writeback" flags could indicate a poisoned page, but Andi [claims](<https://lwn.net/Articles/335779/>) that this approach cannot work. Andrew Morton has [suggested](<https://lwn.net/Articles/335780/>) that HWPOISON could be made into a 64-bit-only feature; Andi allows as to how that might be possible, but he clearly doesn't like the idea. 

Instead, Andi [takes the position](<https://lwn.net/Articles/335781/>) that the page flag shortage does not really exist. It's not a problem at all on 64-bit systems, where `unsigned long` is twice as wide. The number of 32-bit systems with a large number of NUMA nodes is small and shrinking; it's not something that the developers need be concerned about. And, says Andi, if things get really bad, the sparse memory section number can be moved into a separate array like the NUMA node number. Given this view of the problem, worries about adding a useful new feature over concerns about a single page flag bit seem misplaced. 

Nobody has challenged Andi's view that the problem is not as severe as most people think, though Andrew Morton has [hinted](<https://lwn.net/Articles/335783/>) that Andi should go ahead and prove his ideas about moving the section number out of the `page` structure. That might not be a bad idea. Even if page flags are a little more abundant than most developers think, it still is not hard to foresee a time when they are exhausted, at least on 32-bit systems. Proposals involving new page flags are not particularly rare; unless we want to restrict features needing page flags to 64-bit systems, we'll need to make some more flags available before too long.
