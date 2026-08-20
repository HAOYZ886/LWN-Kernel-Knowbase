---
title: BBR congestion control
url: https://lwn.net/Articles/701165/
date: "September 21, 2016"
category: "Networking-Congestion control"
author: "By Jonathan Corbet September 21, 2016"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 21, 2016

Congestion-control algorithms are unglamorous bits of code that allow network protocols (usually TCP) to maximize the throughput of any given connection while simultaneously sharing the available bandwidth equitably with other users. New algorithms tend not to generate a great deal of excitement; the addition of [TCP New Vegas](<https://docs.google.com/document/d/1o-53jbO_xH-m9g2YCgjaf5bK8vePjWP6Mk0rYiRLK-U/edit>) during the 4.8 merge window drew little fanfare, for example. The BBR (Bottleneck Bandwidth and RTT) algorithm just released by Google, though, is attracting rather more attention; it moves away from the mechanisms traditionally used by these algorithms in an attempt to get better results in a network characterized by wireless links, meddling middleboxes, and bufferbloat. 

The problem that any congestion-control algorithm must solve is that the net has no mechanism for informing an endpoint of the bandwidth available for a given connection. So the algorithm must, somehow, come to its own conclusions regarding just how much data it can send at any given time. Since the available bandwidth will generally vary over time, that bandwidth estimate must be revised occasionally. In other words, a congestion control algorithm must maintain an ongoing estimate of how much data can be sent, derived from the information that is available to it. 

That information is somewhat sparse. These algorithms typically work by using one metric that they are easily able to measure: the number of packets that do not make it to the other end of the connection and must be retransmitted. When the network is running smoothly, dropped packets should be a rare occurrence. Once a router's buffers begin to fill, though, it will have no choice but to drop the packets it has no room for. Packet drops are thus a fairly reliable signal that a connection is overrunning the bandwidth available to it and should slow down. 

The problem with this approach, on the network we have now, is that the buffers between any pair of endpoints can be huge. Oversized buffers [have been recognized](<http://bufferbloat.net/>) as a problem for some years now, and progress has been made in mitigating the resulting bufferbloat issues. But the world is still full of bloated routers and some link-level technologies, such as WiFi, require a certain amount of buffering for optimal performance. By the time an endpoint has sent enough data to overflow a buffer somewhere along the connection, the amount of data buffered could be huge. The packet-loss signal, in other words, comes far too late; by the time it is received, an endpoint could have been overdriving the connection for a long time. 

Loss-based algorithms can also run into problems when short-lived conditions cause a dropped packet. They may slow down unnecessarily and, as a result, fail to make use of the bandwidth that is available. 

#### Bottleneck Bandwidth and RTT

The BBR algorithm differs from most of the others in that it pays relatively little attention to packet loss. Instead, its primary metric is the actual bandwidth of data delivered to the far end. Whenever an acknowledgment packet is received, BBR updates its measurement of the amount of data delivered. The sum of data delivered over a period of time is a reasonably good indicator of the bandwidth the connection is able to provide, since the connection has demonstrably provided that bandwidth recently. 

When a connection starts up, BBR will be in the "startup" state; in this mode, it behaves like most traditional congestion-control algorithms in that it starts slowly, but quickly ramps up the transmission speed in an attempt to measure the available bandwidth. Most algorithms will continue to ramp up until they experience a dropped packet; BBR, instead, watches the bandwidth measurement described above. In particular, it looks at the actual delivered bandwidth for the last three round-trip times to see if it changes. Once the bandwidth stops rising, BBR concludes that it has found the effective bandwidth of the connection and can stop ramping up; this has a good chance of happening well before packet loss would begin. 

The measured bandwidth is then deemed to be the rate at which packets should be sent over the connection. But in measuring that rate, BBR probably transmitted packets at a higher rate for a while; some of them will be sitting in queues waiting to be delivered. In an attempt to drain those packets out of the buffers where they languish, BBR will go into a "drain" state, during which it will transmit below the measured bandwidth until it has made up for the excess packets sent before. 

Once the drain phase is done, BBR goes into the steady-state mode where it transmits at more-or-less the calculated bandwidth. That is "more-or-less" because the characteristics of a network connection will change over time, so the actual delivered bandwidth must be continuously monitored. Also, an increase in effective bandwidth can only be detected by occasionally trying to transmit at a higher rate, so BBR will scale the rate up by 25% about 1/8 of the time. If the bandwidth has not increased (transmitting at a higher rate does not result in data being delivered at a higher rate, in other words), that probe will be followed by a drain period to even things out again. 

One interesting aspect of BBR is that, unlike most other algorithms, it does not use the congestion window as the primary means of controlling outgoing traffic. The congestion window limits the maximum amount of data that can be in flight at any given time; an increase in the window will generally result in a burst of packets consuming the newly available bandwidth. BBR, instead, uses the [tc-fq](<http://man7.org/linux/man-pages/man8/tc-fq.8.html>) packet scheduler to send out data at the proper rate. The congestion window is still set as a way of ensuring that there is never too much data in flight, but it is no longer the main regulatory mechanism. 

There is one last complication: many network connections are subject to "policers", middleboxes that limit the maximum data rate any connection can reach. If such a box exists, there is little point in trying to exceed the rate it will allow. The BBR code looks for periods with a suspiciously constant bandwidth (within 4Kb/sec) and a high packet loss rate; should that happen, it concludes that there is a policer in the loop and limits the bandwidth to a level that will not cause that policer to start dropping packets. 

[The BBR patch set](<https://lwn.net/Articles/701177/>) was posted by Neal Cardwell; the code itself carries signoffs from a number of people, including Van Jacobson and Eric Dumazet. Google has, they say, been using BBR for some time, and is evidently happy with the results; BBR works fine when only one side of the connection is using it, so each deployment should, if it lives up to its promises, make the net that much better. We shouldn't have to wait long to find out; networking maintainer David Miller has [applied](<https://lwn.net/Articles/701360/>) the patches, meaning that BBR should be available in the 4.9 kernel.
