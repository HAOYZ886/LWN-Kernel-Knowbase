---
title: The conclusion of the 3.6 merge window
url: https://lwn.net/Articles/509433/
date: "August 3, 2012"
category: "Releases-3.6"
author: "By Jonathan Corbet August 3, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 3, 2012

Linus [closed](<https://lwn.net/Articles/509556/>) the 3.6 merge window on August 2, a couple of days earlier than would have normally been expected. There were evidently two reasons for that: a desire to send a message to those who turn in their pull requests on the last day of the merge window, and his upcoming vacation. In the end, he only pulled a little over 300 changes since [the previous merge window summary](<https://lwn.net/Articles/508790/>), with the result that 8,587 changes were pulled in the 3.6 merge window as a whole. 

Those 300+ changes included the following: 

  * The block I/O bandwidth controller has been reworked so that each control group has its own request list, rather than working from a single, global list. This increases the memory footprint of block I/O control groups, but makes them function in a manner much closer to the original intention when lots of requests are in flight. 

  * A set of [restrictions on the creation of hard and soft links](<https://lwn.net/Articles/503660/>) has been added in an attempt to improve security; they should eliminate a lot of temporary file vulnerabilities. 

  * The device mapper dm-raid module now supports RAID10 (a combination of striping and mirroring). 

  * The list of new hardware support in 3.6 now includes OMAP DMA engines. 

  * The filesystem freeze functionality has been reimplemented to be more robust; in-tree filesystems have been updated to use the new mechanism. 

The process of stabilizing all of those changes now begins; if the usual patterns hold, the final 3.6 kernel can be expected sometime in the second half of September.
