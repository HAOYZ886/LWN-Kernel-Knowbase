---
title: How to use a terabyte of RAM
url: https://lwn.net/Articles/273030/
date: "March 12, 2008"
category: "Block layer-Block drivers; Ramback"
author: "By Jonathan Corbet March 12, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 12, 2008

We have not yet reached a point where systems - even high-end boxes - come with a terabyte of installed memory. But products like those from [Violin Memory](<http://www.violin-memory.com/>) make it clear that the day is coming; one can buy a Violin box with 500GB in it now. So it seems worth asking the question: once one has spent the not inconsiderable sum to buy a box like that, what does one do with all that memory - especially now that the Firefox developers have gotten serious about fixing memory leaks? 

Perhaps it's time for some wild ideas. And there is no better source for such ideas than Daniel Phillips, whose [Ramback patch](<http://lwn.net/Articles/272534/>) has stirred up a bit of discussion this week. The core idea behind Ramback is that all of that memory is turned into a ramdisk, but with a persistent device attached to it. In normal conditions, all application I/O involves only the ramdisk, and is, thus, quite fast (""Every little factor of 25 performance increase really helps.""). In the background, the kernel worries about synchronizing data from the ramdisk onto permanent storage. But the synchronization process is mostly concerned with I/O performance, rather than providing guarantees about just when any given block will make it onto the disk platters. 

Ramback thus differs from the normal block I/O caching done by the kernel in a number of ways. It keeps the entire device in memory, so that, in steady-state operation, applications need never encounter a disk I/O delay. Should an application call `fsync()`, the expected result (blocking until the data is written to physical media) will not happen. Filesystems take great care to order operations in a way that minimizes the risk of data loss in a crash; Ramback ignores all of that and writes data to physical media in whatever order it decides is best. As [Daniel put it](<https://lwn.net/Articles/273032/>), the "most basic principle" of Ramback's design is: 

[T]he backing store is not expected to represent a consistent filesystem state during normal operation. Only the ramdisk needs to maintain a consistent state, which I have taken care to ensure. You just need to believe in your battery, Linux and the hardware it runs on. Which of these do you mistrust? 

Ramback does include an emergency mode which will endeavor to bring the disk up to date in a hurry should the UPS indicate that power has been lost. But that does not seem to be enough for everybody. In the resulting discussion, nobody complained about the sort of performance benefits that a tool like Ramback could provide. But there was a lot of concern about data integrity; it seems that many people distrust their battery, their hardware, _and_ Linux. And that has led to a sort of impasse, with several developers claiming that Ramback would be too risky to use and Daniel dismissing their concerns as FUD. 

FUD or not, those concerns are likely to be a difficult barrier for Ramback to overcome. Meanwhile, Daniel is looking for people to help test out the code, but that presents challenges of its own: 

This driver is ready to try for a sufficiently brave developer. It will deadlock and livelock in various ways and you will have to reboot to remove it. But it can already be coaxed into running well enough for benchmarks, and when it solidifies it will be pretty darn amazing. 

So far, reports from suitably courageous testers have been, well, scarce. Your editor fears that this work could suffer the same fate as many of Daniel's other patches: they can contain brilliant ideas and great coding but just don't quite survive the encounter with the real, messy world. But we need people thinking about how our systems will work in the coming years; one hopes that Daniel won't stop.
