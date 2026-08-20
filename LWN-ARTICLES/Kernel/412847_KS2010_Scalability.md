---
title: "KS2010: Scalability"
url: https://lwn.net/Articles/412847/
date: "November 3, 2010"
category: Scalability
author: "By Jonathan Corbet November 3, 2010 2010 Kernel Summit"
---

By **Jonathan Corbet**  
November 3, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

Scalability is an ongoing challenge for any operating system; the recent shift toward increasing numbers of processors has only made that challenge worse. Paul McKenney led a Kernel Summit session on where we stand with scalability and where things are likely to go. He started with a straw poll asking where the members of the audience thought the logical limit of scalability would be. Most said that 64 CPUs would be the limit, but responses were all across the map from 4096 on the high end to one bowtied joker who said the limit should be zero. 

Paul talked for a little while about the various attributes of systems affecting scalability - NUMAness or the types of peripherals available, for example - and the different kinds of workloads which might be encountered. Scalability means different things in different situations. Linux currently has a long list of concurrency control mechanisms (he showed a full slide) and various data structures which are meant to be scalable. Then he asked: what do we need now? 

James Bottomley asserted that we still have not hit the scalability wall, that the reductionist, lock-splitting approach continues to work. Christoph Hellwig responded that the wall has indeed been hit in the filesystem area, necessitating different approaches. So, for example, they have had to create separate threads to deal with certain kinds of work in a serialized manner. 

Back around 2000 or so, Ted Ts'o said, there was a large, coordinated effort to improve the scalability of the system, then things stopped for a while. People are beginning to pay attention to the problem again, but the work is much more _ad hoc_ , without a visible grand plan. He suggested that all maintainers should be looking at how well their subsystems will work on systems with dozens of cores. Those systems are out there and becoming more common; even if the maintainer doesn't see the problems, users will. Steve Rostedt said that the realtime preemption patches can be a good way to make scalability problems more visible without the need to track down a 64-core system to test kernels on. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

How much farther can scalability work go? Arjan van de Ven fears that we will hit the locking cliff at some point. Ben Herrenschmidt thinks that things are going fairly well, and that improvements in instrumentation have made finding scalability problems easier. There's not much that we have to change in how we are doing things. Tony Luck says that, with 64 cores and 128 threads, every benchmark suffers; there are lots of "little BKLs" and each application hits a different one. It can be hard to find them all before customers do. 

Linus is relatively sanguine about the whole picture. He said that silicon technology is getting closer to the absolute physical limits, so we'll not have to worry about CPU scaling for a whole lot longer. So it's not worth doing a whole lot on the kernel side; unless some wild new technology comes along, there are limits to how much more scalability work will be required. 

[Next: Minisummit reports](<https://lwn.net/Articles/413036/>)
