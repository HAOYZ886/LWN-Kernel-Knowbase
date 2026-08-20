---
title: Being nicer to executable pages
url: https://lwn.net/Articles/333742/
date: "May 19, 2009"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet May 19, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 19, 2009

In an ideal world, our computers would have enough memory to run all of the applications we need. In the real world, our systems are loaded with contemporary desktop environments, office suites, and more. So, even with the large amounts of memory being shipped on modern systems, there still never quite seems to be enough. Memory gets paged out to make room for new demands, and performance suffers. Some help may be on the way in the form of a new [patch](<http://lwn.net/Articles/333489/>) by Wu Fengguang which has the potential to make things better, should it ever be merged. 

The kernel maintains two least-recently-used (LRU) lists for pages owned by user processes. One of these lists holds pages which are backed up by files - they are the page cache; the other list holds anonymous pages which are backed up by the swap device, assuming one exists. When the kernel needs to free up memory, it will do its best to push out pages which are backed up by files first. Those pages are much more likely to be unmodified, and I/O to them tends to be faster. So, with luck, a system which evicts file-backed pages first will perform better. 

It may be possible to do things better, though. Certain kinds of activities - copying a large file, for example - can quickly fill memory with file-backed pages. As the kernel works to recover those pages, it stands a good chance of pushing out other file-backed pages which are likely to be more useful. In particular, pages containing executable code are relatively likely to be wanted in the near future. If the kernel pages out the C library, for example, chances are good that running processes will cause it to be paged back in quickly. The loss of needed executable pages is part of why operations involving large amounts of file data can make the system seem sluggish for a while afterward. 

Wu's patch tries to improve the situation through a fairly simple change: when the page reclaim scanning code hits a file-backed, executable page which has the "referenced" bit set, it simply clears the bit and moves on. So executable pages get an extra trip through the LRU list; that will happen repeatedly for as long as somebody is making use of the page. If all goes well, pages running useful code will stay in RAM, while those holding less useful file data will get pushed out first. It should lead to a more responsive system. 

The code seems to be in a relatively finished state at this point. So one might well ask whether it will be merged in the near future. That is never a straightforward question with memory management code, though. This patch may well make it into the mainline, but it will have to get over some hurdles in the process. The first of those hurdles is [a simple question](<https://lwn.net/Articles/333753/>) from Andrew Morton: 

Now. How do we know that this patch improves Linux? 

Claims like "it feels more responsive" are notoriously hard to quantify. But, without some sort of reasonably objective way to see what benefit is offered by this patch, the kernel developers are going to be reluctant to make changes to low-level memory management heuristics. The fear of regressions is always there as well; nobody wants to learn about some large database workload which gets slower after a patch like this goes in. In summary: knowing whether this kind of patch really makes the situation better is not as easy as one might wish. 

The second problem is that this change would make it possible for a sneaky application to keep its data around by mapping its files with the "executable" bit set. The answer to this objection is easier: an application which seeks unfair advantage by playing games can already do so. Since anonymous pages receive preferable treatment already, the sneaky application could obtain a similar effect on current kernels by allocating memory and reading in the full file contents. Sites which are truly worried about this sort of abuse can (1) use the memory controller to put a lid on memory use, and/or (2) use SELinux to prevent applications from mapping file-backed pages with execute permission enabled. 

Finally, Alan Cox has [wondered](<https://lwn.net/Articles/333758/>) whether this kind of heuristic-tweaking is the right approach in the first place: 

I still think the focus is on the wrong thing. We shouldn't be trying to micro-optimise page replacement guesswork - we should be macro-optimising the resulting I/O performance. My disks each do 50MBytes/second and even with the Gnome developers finest creations that ought to be enough if the rest of the system was working properly. 

Alan is referring to some apparent performance problems with the memory management and block I/O subsystems which crept in a few years ago. Some of these issues [have been addressed](<http://lwn.net/Articles/328363/>) for 2.6.30, but others remain unidentified and unresolved so far. 

Wu's patch will not change that, of course. But it may still make life a little better for desktop Linux users. It is sufficiently simple and well contained that, in the absence of clear performance regressions for other workloads, it will probably find its way into the mainline sooner or later.
