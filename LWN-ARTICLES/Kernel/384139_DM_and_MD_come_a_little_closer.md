---
title: DM and MD come a little closer
url: https://lwn.net/Articles/384139/
date: "April 20, 2010"
category: "Block layer-RAID; RAID"
author: "By Jonathan Corbet April 20, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 20, 2010

The management of RAID arrays in the kernel is a complicated task - and one upon which the fate of much data relies. Given that, it would make sense to have a single set of RAID routines which is improved by all. What the Linux kernel has, instead, is three different RAID implementations: in the multiple device (MD) subsystem, in the device mapper (DM) code, and in the Btrfs filesystem. It has often been said that unifying these implementations would be a good thing, but [that is not easy](<http://lwn.net/Articles/354769/>) and thus far, it has not happened. 

MD maintainer Neil Brown has now taken a step in this direction with the posting of his [dm-raid456 module](<http://lwn.net/Articles/383940/>), a RAID implementation for the device mapper which is built on the MD code. This patch set set has the potential to eliminate a bunch of duplicated code, which can only be a good thing. It also brings some nice features, including RAID6 support, multiple-target support, and more to the device mapper layer. 

This is early work which, probably, is not destined for the next merge window. The response from the device mapper side has been reasonably positive, though. So, with luck, we'll someday have both subsystems using the same RAID code.
