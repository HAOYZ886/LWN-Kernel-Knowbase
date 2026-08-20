---
title: Smarter IRQ suspension in the networking stack
url: https://lwn.net/Articles/1008399/
date: "February 11, 2025"
category: "Networking-NAPI; Releases-6.13"
author: "By Jonathan Corbet February 11, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 11, 2025

High-performance networking is a highly tuned activity; the amount of time available to deal with each packet may be measured in nanoseconds, so care must be taken to avoid anything that might slow the process down. Recently, there has been a fair amount of attention given to [a patch set merged for 6.13](<https://lwn.net/ml/all/20241021015311.95468-1-jdamato@fastly.com/>) that, it is claimed, can improve processing efficiency (and, thus, power savings) in data centers by as much as 30%. The change itself, contributed by Joe Damato and Martin Karsten, is a relatively small tweak to existing optimization techniques; it shows just how much care is needed to optimize a high-bandwidth server. 

In a simple configuration, a network interface will receive a packet, place it in RAM somewhere, then interrupt the CPU to let the kernel know that the packet is ready for processing. That worked well enough when networks were slow, but is decidedly less than optimal in current data centers. Older readers out there will remember a time when every email was special, so we configured our systems to beep (interrupt) at us whenever a message arrived. But that interrupt has a cost, and we do not do that anymore. Instead, we check email occasionally in the sure knowledge that there will be a bountiful supply of new phishing spam waiting for us when we get there. 

The kernel's networking stack has long taken a similar approach. When there is a lot of traffic, the kernel will tell the interface to stop interrupting. Instead, the kernel will poll the interface occasionally to catch up with the new packets that are sure to be waiting. Masking interrupts in this way allows the kernel to process packets in batches, without being interrupted; that helps to improve throughput. 

#### User-space polling

There is room for further improvements, though. By default, the kernel performs this polling in a software-interrupt routine that runs asynchronously from the application that is consuming the incoming data. The interrupt handler and the application will likely often run concurrently; since they are both working on the same network flows, the result can be locking contention and cache misses. If your time budget for processing a packet is measured in nanoseconds, even a single cache miss can cause that budget to be exceeded. 

To address this problem (which only affects the most heavily loaded of servers, but there are a lot of those), the responsibility for polling can be pushed all the way out to user space. If the application selects a special "preferred busy polling" mode, it makes a solemn pledge to the kernel that it will frequently poll the incoming network stream and process the packets that have arrived. The kernel will respond by turning off its own software-interrupt-based packet processing. That processing can, instead, be done when the application polls, so that it will not contend with the application's user-space processing. This kind of polling can yield tiny packet-processing latencies, but it can also drive up CPU usage, especially during times when there are no packets waiting and the application burns CPU time polling without finding any work to do. 

To minimize the CPU-usage problem (and to potentially allow the CPU to go into a lower-power state during slow times), the system can go back to an interrupt-driven mode. If it's not clear when the next packet will arrive, the kernel can simply stop polling, cause the application to block, and request an interrupt instead. There is a tradeoff here, though: in a moderately busy system, there is a good chance that a packet will arrive immediately after the switch to the interrupt-driven mode. Typically, it is better to wait for a little while before doing that. 

To this end, the network stack has a couple of parameters that a high-performance application can tweak. `napi_defer_hard_irqs` is the number of times that an application should be allowed to poll without receiving any data before it is blocked and interrupts are enabled; that will keep the system in the polling mode over tiny gaps in the incoming packet stream. Even after that many attempts, though, interrupts are not enabled immediately; that would invite an interrupt on arrival of the first packet, when there is little work for the interrupt handler to do. It is better to wait a bit longer for traffic to accumulate. So the other parameter, `gro_flush_timeout`, tells the kernel how long it should wait (in nanoseconds) before re-enabling packet-receipt interrupts. 

The `gro_flush_timeout` knob serves a second function as well: is a sort of safety factor, specifying a period of time during which the application should perform at least one poll. If that poll doesn't happen before the timeout period expires, the kernel assumes that the application has gotten distracted and forgotten about its promise to keep polling; it then restarts software-interrupt processing to take the polling responsibility back into its own hands. 

#### A new knob

This dual role for `gro_flush_timeout` is at the root of the problem that was solved by the new patch set. Its value sets a lower bound for the response latency whenever polling stops; if it is set to an overly large value, response times will suffer during slower periods. Pausing for traffic to accumulate is good for throughput, but pausing for too long creates latency. If, instead, this value is set too small, the timeout will trigger while the application is processing packets; that will lead to software-interrupt processing happening concurrently, impacting performance. There is often no value that is perfect for both roles. 

The answer is to split the roles by introducing yet another knob: `irq_suspend_timeout`, which is also specified in nanoseconds. When an application is running in the preferred busy polling mode and receiving data, the value of `irq_suspend_timeout` is used, rather than `gro_flush_timeout`, to determine how long the kernel should wait for the application's next poll before concluding that software-interrupt processing must resume. This timeout will be reset every time the application polls for more data and, importantly, successfully retrieves more data to process. 

The regime changes the moment that a poll returns without finding any data; at that point, the kernel reverts to the older mode, allowing `napi_defer_hard_irqs` empty polls before starting the `gro_flush_timeout` delay, then re-enabling interrupts. In other words, the new timeout only applies while packets continue to arrive. 

This mechanism allows `irq_suspend_timeout` to be set to a relatively long value, since it only applies during busy times when the application is actively processing data. Meanwhile, `gro_flush_timeout`, which only applies when a pause has been seen in incoming traffic, can be set to a relatively short value, with the result that processing will restart quickly once new data arrives. The promised result is both high throughput when traffic is high and low latency when things slow down, while also allowing the CPU to sleep (or perform other work) during those slower times. 

The benchmark results included with the patch set would appear to back up this promise. When running in the new mode, a system is able to deliver consistent (and relatively low) latency as well as if it were running in a full busy-wait mode, but with CPU utilization that is much closer to the full interrupt-deferral case. This is where the claims of power savings come from; a server is able to deliver the required level of service, but without wasting lots of CPU time to contention or doing busy waiting. This one change can, evidently, remove most of the performance advantage that user-space networking solutions can have over the kernel. 

Clearly, this new knob is not going to be something that most users, even those running servers, will want to play with. Enabling preferred busy polling is a balancing act, with a lot of attention required to find the right values for the relevant parameters, and constant monitoring is needed to ensure that the system is running optimally. Adding a new knob makes things a bit more complicated still. But for organizations running unimaginable numbers of servers and trying to get as much performance as possible out of each, this relatively simple tweak to the networking stack could make a world of difference.
