---
title: The end of the 4.7 merge window
url: https://lwn.net/Articles/689257/
date: "June 2, 2016"
category: "Releases-4.7"
author: "By Jonathan Corbet June 2, 2016"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 2, 2016

By the time that Linus released the [4.7-rc1](<https://lwn.net/Articles/689045/>) prepatch and closed the merge window for this development cycle, 10,707 non-merge changesets had been pulled into the mainline repository. As expected, that falls rather short of the 12,172 pulled for 4.6-rc1, but it still adds up to a busy development cycle with a number of interesting changes and new features. 

Some of the changes pulled since [last week's summary](<https://lwn.net/Articles/688217/>) are: 

  * The NFS client now implements the [`copy_file_range()`](<https://lwn.net/Articles/659523/>) system call, making use of the NFS 4.2 `COPY` command to optimize the operation on the remote server. 

  * The direct-access code for persistent memory (DAX) can now work with memory arrays containing media errors. 

  * If the new `TRIM_UNUSED_KSYMS` configuration option is selected, any exported symbols that are not actually used by the built kernel will be removed from the exports list. That might open up more optimization opportunities, and making unused symbols inaccessible seems like a worthwhile change from a security point of view. 

  * A number of longstanding issues with the kernel's string-hashing code, described in [this article](<https://lwn.net/Articles/687494/>), have been addressed through the introduction of a new hashing library. See [`<linux/stringhash.h>`](<https://lwn.net/Articles/689269/>) for the new interface. 

  * New hardware support includes: Sigma Designs "Tango" temperature sensors, thermal sensors attached to analog-to-digital converters, Intel Core SoC power management controllers, Chelsio iSCSI target offload controllers, Texas Instruments TAS5720 mono audio amplifiers, and Maxim MAX98371 codecs. 

If the normal schedule is followed, the 4.7 release can be expected to happen on July 17\. There are no guarantees, of course; that date can be shifted by regressions, unexpected API issues, or irresistible diving opportunities. But the release cycle is predictable enough these days that we can expect that date to not slip by much, if at all. Between now and then, it's just a matter of testing the new kernel and getting the inevitable bugs fixed.
