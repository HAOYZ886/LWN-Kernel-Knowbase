---
title: Fixing the ondemand governor
url: https://lwn.net/Articles/384132/
date: "April 20, 2010"
category: Power management; cpufreq
author: "By Jonathan Corbet April 20, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 20, 2010

The "cpufreq" subsystem is charged with adjusting the CPU clock frequency for optimal performance. Definitions of "optimal" can vary, so there's more than one governor - and, thus, more than one policy - available. The "performance" governor prioritized throughput above all else, while the "powersave" tries to keep power consumption to a minimum. The most commonly-used governor, though, is "ondemand," which attempts to perform a balancing act between power usage and throughput. 

In a simplified form, ondemand works like this: every so often the governor wakes up and looks at how busy the CPU is. If the idle time falls below a threshold, the CPU frequency will be bumped up; if, instead, there is too much idle time, the frequency will be reduced. By default, on a system with high-resolution timers, the minimum idle percentage is 5%; CPU frequency will be reduced if idle time goes above 15%. The minimum percentage can be adjusted in sysfs (under `/sys/devices/system/cpu/cpu _N_ /cpufreq/`); the maximum is wired at 10% above the minimum. 

This governor has been in use for some time, but, as it turns out, it can create performance difficulties in certain situations. Whenever the system workload alternates quickly between CPU-intensive and I/O-intensive phases, things slow down. That's because the governor, on seeing the system go idle, drops the frequency down to the minimum. After the CPU gets busy again, it runs for a while at low speed until the governor figures out that the situation has changed. Then things go idle and the cycle starts over. As it happens, this kind of workload is fairly common; "git grep" and the startup of a large program are a couple of examples. 

Arjan van de Ven has come up with [a fix for this governor](<http://lwn.net/Articles/383838/>) which is quite simple in concept. The accounting of "idle time" is changed so that time spent waiting for disk I/O no longer counts. If a processor is dominated by a program alternating between processing and waiting for disk operations, that processor will appear to be busy all the time. So it will remain at a higher frequency and perform better. That makes the immediate problem go away without, says Arjan, significantly increasing power consumption. 

But, Arjan [says](<https://lwn.net/Articles/384134/>), ""there are many things wrong with ondemand, and I'm writing a new governor to fix the more fundamental issues with it"". That code has not yet been posted, so it's not clear what sort of heuristics it will contain. Stay tuned; the demand for ondemand may soon be reduced significantly.
