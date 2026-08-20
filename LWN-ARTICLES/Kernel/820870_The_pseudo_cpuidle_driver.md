---
title: The pseudo cpuidle driver
url: https://lwn.net/Articles/820870/
date: "May 21, 2020"
category: "Development tools; Power management-cpuidle"
author: "By Jonathan Corbet May 21, 2020 OSPM"
---

By **Jonathan Corbet**  
May 21, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

The purpose of a cpuidle governor is to decide which idle state a CPU should go into when it has no useful work to do; the cpuidle _driver_ then actually puts the CPU into that state. But, at the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), Abhishek Goel presented a new cpuidle driver that doesn't actually change the processor's power state at all. Such a driver will clearly save no power, but it can be quite useful as a tool for evaluating and debugging cpuidle policies. 

Goel began by saying that this work was motivated by a performance problem encountered with high-latency idle states — deep CPU sleep states that take a long time to restart from. A GPU-oriented workload that generated lots of interrupts was involved; the time between those interrupts was just enough to cause the governor to choose a deep idle state. That created latency which added up over time as the workload progressed. The temporary workaround was to increase the target latency (the expected sleep time) for those idle states by a factor of three to five, biasing the idle-state choice toward the shallower states. It solved the problem, but is "not elegant"; others will undoubtedly find workloads that go wrong in other ways. 

Rafael Wysocki interjected to suggest using the [pm_qos mechanism](<https://lwn.net/Articles/386139/>) instead; its purpose is to address just this sort of latency issue, and he was curious to know why it didn't work. Part of the problem, evidently, is that pm_qos will disable the deeper idle states entirely, but there can be value in having them remain available for the truly long idle periods. Parth Shah added that, [![\[Abhishek Goel\]](https://static.lwn.net/images/conf/2020/ospm/AbhishekGoel2-sm.png)](<https://lwn.net/Articles/820871/>) on the Power architecture, this is even more true; without getting to those deeper idle states little energy will be saved. 

Goel tried providing a debugfs interface for the cpuidle core that would allow the residency attributes of the various idle states to be tweaked at run time. It is useful for validating idle states, he said, but synchronization of the changes within the kernel is painful. Changing these attributes can also lead to strange behavior due to distortions of the CPU's idle history. He wanted a better solution. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The result was the pseudo cpuidle driver. It is a loadable module that allows the user to create customized idle states and tweak the attributes as they go. Doing things this way avoids both the synchronization and history-distortion problems. The module is loaded with a set of parameters describing the number of states, along with the target residency and exit latency of each. The actual "idle states" are implemented by putting the CPU into a busy-wait loop, spinning until the next wakeup event happens; the governor then spins a bit longer to simulate the exit latency time. 

This behavior clearly does a poor job of saving power, but it is useful to evaluate how specific policies affect system performance. It can be used for tuning a governor, or to compare the effects of different governors entirely. It also turns out to be useful for CPU design, he said; designers can try out various idle states and see how they will actually perform. 

Future work, he concluded, could include simulating idle states at the core and chip levels as well as basic CPU states. He is also planning to add some tracing capabilities to the driver. 

Wysocki led off the discussion by pointing out one potential problem. In a real system, the response to a hardware interrupt will be delayed by the exit latency of the processor. The only way to simulate that delay would be to do the busy-waiting with interrupts disabled, but then the interrupt (which would normally wake the system) will be missed entirely. That particular aspect of hardware behavior, it seems, cannot be emulated in this way. That said, he agreed that the driver looks useful for studying cpuidle governors, and seems worth having. 

At the conclusion of the session, Juri Lelli asked if there were any sort of performance numbers comparing this driver with real hardware; Goel answered that he didn't have those yet.
