---
title: "CPUS*PIDS = mess"
url: https://lwn.net/Articles/384936/
date: "April 27, 2010"
category: Scalability
author: "By Jonathan Corbet April 27, 2010"
---

By **Jonathan Corbet**  
April 27, 2010

Mike Travis recently [ran into a problem](<https://lwn.net/Articles/384937/>): if you have a system with a mere 2048 processors, there's only room for 16 processes on each CPU before the default 32K limit on process IDs is reached. Systems with lots of processors tend not to run large numbers of processes on each CPU, but 16 is still a bit tight - especially when one considers how many kernel threads run on each CPU. With 2K processors, the kernel threads alone may run the system out of process IDs; with 4K processors, the system will not even succeed in booting. 

The proposed solution was a new boot-time parameter allowing the specification of a larger maximum number of process IDs. That idea did not get very far, though; there is not much interest in adding more options just to enable the system to boot. The fact that [concurrency-managed workqueues](<http://lwn.net/Articles/355700/>) should eventually solve this problem (by getting rid of large numbers of workqueue threads) hasn't helped either; that makes the kernel option look like a temporary stopgap. But the workqueue changes are only so helpful to people who are having this problem now; some form of this work will probably go in eventually, but it does not appear to be a fast process. 

So there will most likely be a shorter-term fix merged. Instead of a kernel parameter, though, it will probably be some sort of heuristic which looks at the number of processors and ensures that a sufficient number of process IDs is available. If the default limit is too low, it will be raised automatically. 

There is one remaining concern: what about ancient applications which store process IDs in signed, 16-bit integers? Apparently such applications exist. It is less clear, though, that such applications exist on 4096-processor systems. So this fear is unlikely to hold up this change. By the time the rest of us get those shiny, new, 4096-core desktop systems, hopefully, any remaining broken applications will have long since been fixed.
