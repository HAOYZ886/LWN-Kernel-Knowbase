---
title: "NAPI performance - a weighty matter"
url: https://lwn.net/Articles/139884/
date: "June 15, 2005"
category: "Interrupts; NAPI; Networking-NAPI"
author: 
---

Modern network interfaces are easily capable of handling thousands of packets per second. They are also capable of burying the host processor under thousands of interrupts per second. As a way of dealing with the interrupt problem (and fixing some other things as well), the networking hackers added the NAPI driver interface. NAPI-capable drivers can, when traffic gets high, turn off receive interrupts and collect incoming packets in a polling mode. Polling is normally considered to be bad news, but, when there is always data waiting on the interface, it turns out to be the more efficient way to go. Some details on NAPI can be found in [this LWN Driver Porting Series article](<http://lwn.net/Articles/30107/>); rather more details are available from the networking chapter in [LDD3](<https://lwn.net/Kernel/LDD3/>). 

One of the things NAPI-compliant drivers must do is to specify the "weight" of each interface. The weight parameter helps to determine how important traffic from that interface is - it limits the number of packets each interface can feed to the networking core in each polling cycle. This parameter also controls whether the interface runs in the polling mode or not; by the NAPI conventions, an interface which does not have enough built-up traffic to fill its quota of packets (where the quota is determined by the interface's weight) should go back to the interrupt-driven mode. The weight is thus a fundamental parameter controlling how packet reception is handled, but there has never been any real guidance from the networking crew on how the weight should be set. Most driver writers pick a value between 16 and 64, with interfaces capable of higher speeds usually setting larger values. 

Some recent discussions on the netdev list have raised the issue of how the weight of an interface should be set. In particular, the e1000 driver hackers have discovered that their interface tends to perform better when its weight is set lower - with the optimal value being around 10. Investigations into this behavior continue, but a few observations have come out; they give a view into what is really required to get top performance out of modern hardware. 

One problem, which appears to be specific to the e1000, is that the interface runs out of receive buffers. The e1000 driver, in its `poll()` function, will deliver its quota of packets to the networking core; only when that process is complete does the driver concern itself with providing more receive buffers to the interface. So one short-term tactic would be to replenish the receive buffers more often. Other interface drivers tend not to wait until an entire quota has been processed to perform this replenishment. Lowering the weight of an interface is one way to force this replenishment to happen more often without actually changing the driver's logic. 

But questions remain: why is the system taking so long to process 64 packets that a 256-packet ring is being exhausted? And why does performance increase for smaller weights even when packets are not being dropped? One possible explanation is that the actual amount of work being done for each packet in the networking core can vary greatly depending on the type of traffic being handled. Big TCP streams, in particular, take longer to process than bursts of small UDP packets. So, depending on the workload, processing one quota's worth of packets might take quite some time. 

This processing time affects performance in a number of ways. If the system spends large bursts of time in software interrupt mode to deal with incoming packets, it will be starving the actual application for processor time. The overall latency of the system goes up, and performance goes down. Smaller weights can lead to better interleaving of system and application time. 

A related issue is this check in the networking core's polling logic: 

```
if (budget <= 0 || jiffies - start_time > 1)
    		goto softnet_break;
```

Essentially, if the networking core spends more than about one half of one jiffy (very approximately 500 μsec on most systems) polling interfaces, it decides that things have gone on for long enough and it's time to take a break. If one high-weight interface is taking a lot of time to get its packets through the system, the packet reception process can be cut short early, perhaps before other interfaces have had their opportunity to deal with their traffic. Once again, smaller weights can help to mitigate this problem. 

Finally, an overly large weight can work against the performance of an interface when traffic is at moderate levels. If the driver does not fill its entire quota in one polling cycle, it will turn off polling and go back into interrupt-driven mode. So a steady stream of traffic which does not quite fill the quota will cause the driver to bounce between the polling and interrupt modes, and the processor will have to handle far more interrupts that would otherwise be expected. Slower interfaces (100 Mb/sec and below) are particularly vulnerable to this problem; on a fast system, such interfaces simply cannot receive enough data to fill the quota every time. 

From all this information, some conclusions have emerged: 

  * There needs to be a smarter way of setting each interface's weight; the current "grab the setting from some other driver" approach does not always yield the right results. 

  * The direct tie between an interface's weight and its packet quota is too simple. Each interface's quota should actually be determined, at run time, by the amount of work that interface's packet stream is creating. 

  * The quota value should not also be the threshold at which drivers return to interrupt-driven mode. The cost of processor interrupts is high enough that polling mode should be used as long as traffic exists, even when an interface almost never fills its quota. 

Changing the code to implement these conclusions is likely to be a long process. Fundamental tweaks in the core of the networking code can lead to strange performance regressions in surprising places. In the mean time, Stephen Hemminger has posted [a patch](<http://lwn.net/Articles/138781/>) which creates a sysfs knob for the interface weight. That patch has been merged for 2.6.12, so people working on networking performance problems will soon be able to see if adjustable interface weights can be part of the solution.
