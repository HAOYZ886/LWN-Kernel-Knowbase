---
title: Laptop mode for 2.6
url: https://lwn.net/Articles/65437/
date: "January 7, 2004"
category: Block layer; Laptop mode; Power management
author: 
---

Some months ago, Jens Axboe posted a "laptop mode" patch for the 2.4 kernel. That patch had never been ported forward to 2.6, until now. Bart Samwel has picked up the laptop mode baton and posted several versions of a 2.6 patch; the latest, as of this writing, is [version 6](<https://lwn.net/Articles/65438/>). 

The purpose of the patch is to allow laptop users to get the greatest amount of time out of their batteries by minimizing the time the disk spends spinning. Any Linux conference attendee who has ever lost the race for the available power outlets can't help but appreciate this idea. To keep the disk idle, the patch (along with an associated script) changes system behavior in the following ways: 

  * The amount of time the system is willing to wait before writing dirty pages to disk is expanded to ten minutes. As a result, laptop mode users risk losing up to ten minutes worth of work, but that is a risk many will be willing to take. 

  * Any ext3 or ReiserFS filesystems will be remounted with a commit period of ten minutes. 

  * Background writeback of dirty pages, normally done when the disk is not busy doing anything else, is disabled. 

  * When something does force the disk to spin up, the system writes out all dirty pages regardless of how long they have been in memory. In this way, the kernel tries to accomplish all the work it can during the brief time that the disk is spinning. 

There is also a separate mode which can be enabled which creates a log message every time a process forces some disk activity. This feature is useful for solving those "why is the disk spinning up" mysteries. An older version of the laptop mode patch is currently in the 2.6.1-rc1-mm2 tree, which suggests that it may yet find its way into a 2.6 kernel. Thousands of power-starved laptop users will be grateful.
