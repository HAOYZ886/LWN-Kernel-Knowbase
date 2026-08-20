---
title: Tightening the merge window rules
url: https://lwn.net/Articles/297529/
date: "September 9, 2008"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet September 9, 2008"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 9, 2008

The [2005 kernel summit](<http://lwn.net/Articles/144281/>) included a discussion on a recurring topic: how can the community produce kernels with fewer bugs? One of the problems which was identified in that session was that significant changes were often being merged late in the development cycle with the result that there was not enough time for testing and bug fixing. In response, the summit attendees proposed the concept of the "merge window," a two-week period in which all major changes for a given development cycle would be merged into the mainline. Once the merge window closed, only fixes would be welcome. 

Three years later, the merge window is a well established mechanism. Over that time, the discipline associated with the merge window has gotten stronger; it is now quite rare that significant changes go into the mainline outside of the merge window. The one notable exception is that new drivers can be accepted later in the cycle, based on the reasoning that a driver, being completely new and self-contained functionality, cannot cause regressions. Even then, there are hazards: the UVC webcam driver, merged quite late in the 2.6.26 cycle (in 2.6.26-rc9), brought a security hole with it. 

The merge window rule is often expressed as "only fixes can go in after the -rc1 release." Recent discussions have made it clear, though, that Linus is starting to develop a rather more restrictive view of how development should go outside of the merge window. The imminent 2008 kernel summit may well find itself taking on this topic and making some changes to the rules. 

In short, Linus has concluded that "fixes only" is not disciplined enough; a lot of work characterized as a "fix" can, itself, be a source of new regressions. So here's [how Linus would like developers to operate](<https://lwn.net/Articles/297533/>) now: 

Here's a simple rule of thumb: 

  * if it's not on the regression list 
  * if it's not a reported security hole 
  * if it's not on the reported oopses list 
then why are people sending it to me? 

There can be no doubt that the tighter rules have come as a surprise to a number of developers - if nothing else, the frequency with which Linus has found himself getting grumpy with patch submitters makes that clear. 

And, the truth of the matter is that Linus has not enforced anything like the above rule in the past. Beyond new drivers, post-merge-window changes have typically included things like coding style and white space fixups, minor feature enhancements, defconfig updates, documentation updates, annotations for the sparse tool, and so on. Relatively few of these changes come equipped with an entry on the regression list. 

To look at this another way, here's a table which appeared in [the 2.6.26 development statistics](<http://lwn.net/Articles/288233/>) article, updated with 2.6.27 (to date) information: 

> Release| Changesets merged  
> ---|---  
> For -rc1| after -rc1  
> 2.6.23| 4505| 2570  
> 2.6.24| 7132| 3221  
> 2.6.25| 9629| 3078  
> 2.6.26| 7555| 2577  
> 2.6.27*| 7733| 2451  
> * (Through September 9). 

2.6.27 appears to be following the trend set by previous kernels: on the order of 25% of the total changesets will be merged outside of the nominal merge window. The most recent [2.6.27 regression summary](<http://lwn.net/Articles/297275/>) shows a total of 150 regressions during this development cycle, of which 33 were unresolved. That suggests that at least 2300 patches merged since 2.6.27-rc1 were not fixes for listed regressions. 

So the "regression fixes only" policy is truly new - and not really effective yet. Should this policy hold, it could have a number of interesting implications including, perhaps, an increase in the number of non-regression fixes shipped in distributor kernels. It might make developers become more diligent about reporting regressions so that the associated fix can be merged. With fewer changes going in later in the cycle, development cycles might just get a little shorter, perhaps even to the eight weeks that was, once, the nominal target. And, of course, we might just get kernel releases with fewer bugs, which would be a hard thing to complain about. In the short term, though, expect more grumpy emails to developers who are still trying to work by the older rules.
