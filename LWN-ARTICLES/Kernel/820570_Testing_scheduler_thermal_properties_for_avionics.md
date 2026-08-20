---
title: Testing scheduler thermal properties for avionics
url: https://lwn.net/Articles/820570/
date: "May 15, 2020"
category: Benchmarking; Thermal management
author: "By Jonathan Corbet May 15, 2020 OSPM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 15, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

Linux is not heavily used in safety-critical systems — yet. There is an increasing level of interest in such deployments, though, and that is driving a number of initiatives to determine how Linux can be made suitable for safety-critical environments. At the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), Michal Sojka shone a light on one corner of this work: testing the thermal characteristics of Linux systems with an eye toward deployment in avionics systems. 

In particular, his focus is on how scheduling decisions can affect the thermal behavior of computers in avionic systems; this effort is part of the European [THERMAC project](<https://cordis.europa.eu/project/id/832011>). The requirements for avionic systems include doing without both fans and heavy heat sinks while getting as much performance out of each system as thermal constraints will allow. There is no room for missed deadlines in safety-critical work, so there is not much space for the usual thermal-management techniques there. But these systems also support best-effort workloads that run when time and temperatures allow; that is where it may be possible to improve the situation with clever power management. 

These systems tend to use time-partitioned scheduling. Each safety-critical task runs within its own time window; any time left over within the window when that work is done can be used for best-effort workloads. The good news, Sojka said, was that the workloads on these systems are well understood; that is a distinct difference from the systems discussed in [the previous session](<https://lwn.net/Articles/820432/>), where the kernel has to make guesses about what is going to happen next. 

This work, so far, has not yet come up with any thermal-aware scheduling strategies; that is for a later stage. What is being done now is to put together the framework for evaluating such strategies so it will be possible to know which ones actually work. To that end, the project has built a testbed based on a leading-edge NXP i.MX8 board; thermal sensors and a thermal camera have been added to that. Control groups are being used to simulate the scheduling windows that will be used on a real system. 

The work so far has resulted in a framework called "[thermobench](<https://github.com/CTU-IIG/thermobench>)"; Sojka described it as "a fancy CSV file generator". It will run a series of benchmarks, capturing measurements (temperatures, CPU frequencies, CPU [![\[Michal Sojka\]](https://static.lwn.net/images/conf/2020/ospm/MichalSojka-sm.png)](<https://lwn.net/Articles/820571/>) loads, etc.) as they go. When the runs are complete, the system can create plots of what happened. The benchmarks in the repository now include various micro-instruction tests and tests that evaluate a variety of sleep patterns. 

The system can also perform model fitting in order to get a sense for the changes that happen at different time scales; some changes happen much more quickly than others, leading to a model equation with three distinct terms. The temperature at the heat sink can change within a minute, while whole-board temperature changes play out over four or five minutes. There is also an 18-minute term which, he surmised, was the response of the entire testbed. Among other things, these results tell them how long each test needs to run for. 

In conclusion, he said, thermobench will be useful for comparing various thermal management strategies. He wondered whether others might find it useful for their areas as well. Vincent Guittot asked whether the tests included CPU-frequency scaling; Sojka answered the tests that were shown are all single-frequency tests, but multiple-frequency tests have been done as well. He said that temperature is not a linear function of CPU frequency, but did not get into details. 

Rafael Wysocki said that the tests should always measure both the power consumption of the board and the temperature, since the two are somewhat independent of each other. Giovanni Gherdovich asked whether the realtime preemption patches had been tested, noting that kernels with those patches have different performance and power-usage profiles. Sojka answered that the test board is quite new and is currently not able to run a mainline kernel; he expressed interest in hearing what NXP's plans are for getting support upstream. Once that happens, he will be happy to experiment with the realtime patches. 

Souvik Chakravarty pointed out that a number of factors affect power usage. For example, what is the power structure of the board? If all CPUs are on a single power rail, it will be necessary to stop them all to gain significant power (and thermal) savings. Sojka said that the processor in question has six [big.LITTLE](<https://en.wikipedia.org/wiki/ARM_big.LITTLE>) CPUs, and the project is testing on the little CPUs only. But details like the power layout are not entirely clear. 

Sojka concluded by encouraging attendees to check out the thermobench code, which had been posted that very day.
