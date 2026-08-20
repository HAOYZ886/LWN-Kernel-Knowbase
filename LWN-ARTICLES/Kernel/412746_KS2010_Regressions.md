---
title: "KS2010: Regressions"
url: https://lwn.net/Articles/412746/
date: "November 2, 2010"
category: "Development model-Regressions"
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

For the last few years, Rafael Wysocki has been the main tracker of regressions in the Linux kernel. Avoiding regressions is important: if the kernel is allowed to regress, there can never be any assurance that things are getting better over time. So it's not surprising that, as with previous years, regressions were a topic at the 2010 Kernel Summit and that the developers in attendance were interested. 

Rafael started by pointing out that regression tracking is no longer a one-person show. Entries in the kernel bugzilla are now usually created by Maciej Rutecki, while the maintenance of bug reports is done by Florian Mickler. Andrew Morton does the work of noticing new bug reports in the first place, and Rafael generates and posts the lists of regressions. All told, the system seems to work well with less stress on any single person. 

A number of tables full of numbers were posted; your editor could not possibly have copied them all down. The good news is that we hope to run an article on regressions written by Rafael himself in the very near future; that article should more than fill in any gaps in the data. Meanwhile, here are the regression numbers for the last year or so of kernel releases: 

> Kernel | Regressions  
> ---|---  
> Reported| Unfixed  
> 2.6.31| 146| 20  
> 2.6.32| 133| 28  
> 2.6.33| 116| 18  
> 2.6.34| 119| 15  
> 2.6.35| 63| 28  
  
It may look like the situation began to improve around 2.6.33, but there's an important thing to note: the way regressions are recorded was changed at that time. Prior to 2.6.33, regressions were added to the list as soon after they were reported; starting in 2.6.33, they will only be added if they remain unfixed for at least one week. Rafael says there was little point in recording regressions which vanish almost immediately. Once this change is taken into account, he said, the regression numbers have not actually changed much over this time. 

He put up another table showing when regressions were reported. Through 2.6.32, the number of unfixed regressions in 2.6.n was higher when 2.6.n+1 was released than when 2.6.n came out. After that time, the situation reversed; the number of unfixed regressions in a given kernel release will be lower when the subsequent release comes out. So, it seems, we are getting a little better about fixing things. 

How much better is not clear, though. According to Rafael, the mean lifetime of a reported regression is "approximately 24.5 days." That number has not really changed over the last two years. 

Where are the regressions being found? The direct rendering subsystem is by far the biggest offender, with the Intel-specific DRI code, on its own, being at the top of the list. After that are the x86 architecture, the filesystem code, and the network subsystem. This information led to the fairly obvious conclusion that regressions tend to be found in code which (1) is actually used, and (2) is under relatively heavy development. Rafael also noted that code which deals closely with hardware tends to be more prone to regressions than core kernel code. 

What about regressions in the stable tree? Those should not happen at all, but they do anyway. One reason for that is that code in stable releases tends to get very good test coverage after the release. But regressions also happen because there is real value in aggressively pushing fixes into the stable tree. Rafael suggested that the stable tree needs a rule to the effect that any patch causing a regression will be instantly reverted; Greg Kroah-Hartman responded that there is already such a rule. It seems, though, that Greg isn't always being informed when stable-tree regressions are reported. 

Steve Rostedt asked how many fixes for regressions actually cause regressions of their own. Thomas Gleixner retorted that such things only happen in the tracing code. Rafael, more seriously, noted that it does happen, but he doesn't track those occurrences. 

Rafael took some time to complain about the state of the kernel bugzilla system which, it seems, is essentially unmaintained at the moment. Sometimes it just doesn't work, it doesn't track information he would like to have, an account is required for anybody who would like to be on the Cc list, etc. So it seems that there is an opening available for somebody who would like to volunteer to improve the kernel bugzilla; volunteers were notably absent from the room, though. 

[Next: Performance regressions](<https://lwn.net/Articles/412747/>).
