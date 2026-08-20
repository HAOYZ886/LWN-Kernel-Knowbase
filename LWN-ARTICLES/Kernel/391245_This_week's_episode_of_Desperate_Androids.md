---
title: "This week's episode of \"Desperate Androids\""
url: https://lwn.net/Articles/391245/
date: "June 7, 2010"
category: Android
author: "By Jonathan Corbet June 7, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 7, 2010

In last week's episode of this particular soap opera, we [described](<http://lwn.net/Articles/390369/>) the criticism that blocked the merging of suspend blockers and the shape of a solution which was seemingly emerging from the ruins. But declaring an end to this particular story is always a hazardous thing to do. Now it looks like the suspend blocker discussion may last for another release cycle or two. 

On June 4, Ingo Molnar [posted](<https://lwn.net/Articles/391247/>) a proposed solution which was a variant of the quality-of-service-based approach described last week. It looked feasible until Linus [wandered into the discussion](<https://lwn.net/Articles/391248/>) (for the first time), saying: 

Quite frankly, this sounds fundamentally broken. Think deadlock. The high-latency task got a lock, and now you're excluding it because it scheduled away. 

So the discussion started anew. Linus pushed for a solution which avoids the rest of the core kernel (and the scheduler in particular) as much as possible - a goal which the initial suspend blocker implementation shared. He also raised the issue of multi-core processors, which are not really addressed by the current code. There might be value in being able to shut down individual cores as the system load drops, suspending the system when there's nothing for the last CPU to do. One assumes that SMP handsets are not that far away, so planning for them would be a sensible thing to do. 

All told, the situation has grown more complicated - but it also seems that the will to solve it has grown. It is becoming clear that the real solution may not show up in a hurry, though. So, in the meantime, we may see a stopgap solution which was first proposed early in the discussion: add stub versions of the suspend blocker API so that various Android drivers can be merged unchanged. That should help the mainline and Android kernels come much closer to convergence while allowing time for a globally acceptable solution to the suspend blocker problem to be solved. We will likely see those stubs merged, possibly with a 2.6.37 expiration date. The more contentious stuff will come some time after.
