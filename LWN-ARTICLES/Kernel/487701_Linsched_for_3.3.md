---
title: Linsched for 3.3
url: https://lwn.net/Articles/487701/
date: "March 21, 2012"
category: Development tools; Scheduler
author: "By Jonathan Corbet March 21, 2012"
---

By **Jonathan Corbet**  
March 21, 2012

At the [2011 Kernel Summit](<https://lwn.net/Articles/464296/>), Google developer Paul Turner described a scheduler testing framework which, he said, would be released soon. Naturally, things took longer than expected, but, on March 14, Paul [released](<https://lwn.net/Articles/486635/>) a version of Linsched for general use. Given the amount of interest in this tool, it's likely that it will find its way into the mainline in a relative hurry. 

Linsched is a framework that can run the kernel scheduler with various simulated workloads and draw conclusions about the quality of the decisions made. It looks at overall CPU utilization, the number of migrations, and more. It is able to simulate a wide range of hardware topologies with different characteristics. 

The original Linsched posting was quite intrusive; it inserted over 5,000 lines of code into the kernel behind "`#ifdef LINSCHED`" lines. A determined effort has reduced that number slightly - to all of 20 lines of code. The rest has been cleverly hidden in a special "linsched" architecture that provides just enough support to run the scheduler in user space. The actual simulation and measurement code lives in the `tools` directory. 

Making changes to the scheduler is a notoriously difficult task; one can easily add regressions for specific workloads that go unnoticed until the changes go into production. With enough simulated topologies and workloads, a tool like Linsched should be able to remove a lot of that risk from scheduler development. And that should lead to better kernel releases overall.
