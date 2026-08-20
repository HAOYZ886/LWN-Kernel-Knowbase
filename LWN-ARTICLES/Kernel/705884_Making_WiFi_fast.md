---
title: Making WiFi fast
url: https://lwn.net/Articles/705884/
date: "November 8, 2016"
category: "Networking-Wireless"
author: "By Jonathan Corbet November 8, 2016 Linux
Plumbers Conference"
---

By **Jonathan Corbet**  
November 8, 2016

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2016-Linux_Plumbers_Conference>)

Dave Täht has been working to save the Internet for the last six years (at least). Recently, his focus has been on improving the performance of networking over WiFi — performance that has been disappointing for as long as anybody can remember. The good news, as related in his [2016 Linux Plumbers Conference](<https://www.linuxplumbersconf.org/2016/>) talk, is that WiFi can be fixed, and the fixes aren't even all that hard to do. Users with the right hardware and a willingness to run experimental software can have fast WiFi now, and it should be available for the rest of us before too long. 

Networking, Täht said, has been going wrong for over a decade; it turns out that queuing theory has not properly addressed the problem of matching data rates to the bandwidth that the hardware can provide. Developers have tended to optimize for the fastest rates possible, but those rates are rarely seen in the real world when WiFi is involved. The "make WiFi fast" effort, involving a number of developers, seeks to change the focus and to optimize both throughput and latency at all data rates. 

He has been working on the [bufferbloat](<https://www.bufferbloat.net/projects/>) problem for the last six years. Hundreds of people have been involved in this effort, which was spearheaded by the Linux networking stack. Many changes were merged, starting with [byte queue limits](<https://lwn.net/Articles/454390/>) in 3.3 and culminating (so far) with [the BBR congestion-control algorithm](<https://lwn.net/Articles/701165/>), which was merged for 4.8. At this point, all network protocols can be debloated — with the exception of WiFi and LTE. But, he said, a big dent has just been made in the WiFi problem. 

For the rest of the talk, Täht enlisted the aid of Ham the mechanical monkey. Ham, it seems, works in the marketing department. He only cares about benchmarks; if the numbers are big, they will help to sell products. [![\[Dave Täht\]](https://static.lwn.net/images/conf/2016/lpc/DaveTaht-sm.jpg)](<https://lwn.net/Articles/705911/>) Ham has been the nemesis for years, driving the focus in the wrong direction. The right place to focus is on use cases, where the costs of bufferbloat are felt. That means paying much more attention to latency, and focusing less on the throughput numbers that make Ham happy. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

As an example, he noted that the Slashdot home page can, when latency is near zero, be loaded in about eight seconds (the LWN page, he said, was too small to make an interesting example). If the [Flent](<https://flent.org/>) tool is used to add one second of latency to the link, that load takes nearly four minutes. We have all been in that painful place at one point or another. The point is that latency and round-trip times matter more than absolute throughput. 

Unfortunately, the worst latency-causing bufferbloat is often found on high-rate connections deep within the Internet service provider's infrastructure. That, he said, should be fixed first, and WiFi will start to get better for free. But that is only the start. WiFi need not always be slow; its problems are mostly to be found in its queuing, not in external factors like radio interference. The key is eliminating bufferbloat from the WiFi subsystem. 

To get there, Täht and his collaborators had to start by developing a better set of benchmarks to show what is going on in real-world situations. The most useful tool, he said, is Flent, which is able to do repeatable tests under network load and show the results in graphical form. Single-number benchmark results are not particularly helpful; one needs to look at performance over time to see what is really going on. It is also necessary to get out of the testing lab and test in the field, in situations with lots of stations on the net. 

What they found was that the multiple-station case is where things fall down in the WiFi stack. If you have a single device on a WiFi network, things will work reasonably well. But as soon as there is contention for air time, the problems show up. 

#### How to improve WiFi

The WiFi stack in current kernels has four major layers of interest, when it comes to queuing: 

  * At the top, the queuing discipline accepts packets and feeds them into the driver layer. The amount of buffering there is huge; it can hold ten seconds of WiFi data. 

  * The mac80211 layer does high-level WiFi work, and adds some queuing and latency of its own. 

  * The driver for the WiFi adapter maintains several queues of its own, perhaps holding several seconds of data. This level is where aggregation is done; aggregation groups a set of packets into a single transmitted frame to improve throughput — at the cost of increased latency. 

  * The firmware in the adapter itself can hold another ten seconds of data in its queues. 

That adds up to a lot of queuing in the WiFi subsystem, with all of the associated problems. The good news is that fixing it required no changes to the WiFi protocols at all. So those fixes can be applied to existing networks and existing adapters. 

The first step was to add a "mac80211 intermediate queue" that handles all packets for a given device, reducing the amount of queuing overall, especially since the size of this queue is strictly limited. It is meant to to hold no more data than can be sent in two "transmission opportunities" (slots in which an aggregate of packets can be transmitted). The fq_codel queue management algorithm was generalized to work well in this setting. 

The queuing discipline layer was removed entirely, eliminating a massive amount of buffering. Instead, there is a simple per-station queue, and round-robin fair queuing between the stations. The goal is to have one aggregated frame in the hardware for transmission, and another one queued, ready to go as soon as the hardware gets to it. Only having two packets queued at this layer may not scale to the very highest data rates, he said, but, in the real world, nobody ever sees those rates anyway. 

There should be a single aggregate under preparation in the mac80211 layer; all other packets should be managed in the (short) per-station queues. In current kernels, mac80211 pushes packets into the low-level driver, where they may accumulate. In the new model, instead, the driver calls back into the mac80211 layer when it needs another packet; that gives mac80211 a better view into when transmission actually happens. The total latency imposed by buffering in this scheme is, he said, limited to 2-12ms, and there is no need for intelligence in the network hardware. 

#### Results and future directions

The result of all this work is WiFi latencies that are less than 40ms, down from a peak of 1-2 seconds before they started, and much better handling of multiple stations running at full rate. Before the changes, a test involving 100 flows all starting together collapsed entirely, with at most five flows getting going; all the rest failed due to TCP timeouts caused by excessive buffering latency. Afterward, all 100 could start and run with reasonable latency and bandwidth. All this work, in the end, comes down to a patch that removes a net 200 lines of code. 

There are some open issues, of course. The elimination of the queuing discipline layer took away a number of useful network statistics. Some of these have been replaced with information in the debugfs filesystem. There is, he said, some sort of unfortunate interaction with [TCP small queues](<https://lwn.net/Articles/507065/>); Eric Dumazet has some ideas for fixing this problem, which only arises in single-station tests. There is an opportunity to add better air-time fairness to keep slow stations from using too much transmission time. Some future improvements, he said, might come at a cost: latency improvements might reduce the peak bandwidth slightly. But latency is what almost all users actually care about, so that bandwidth will not be missed — except by Ham the monkey. 

At this point, the ath9k WiFi driver fully supports these changes; the code can be found in the [LEDE](<https://lede-project.org/>) repository and daily snapshots. Work is progressing on the ath10k driver; it is nearly done. Other drivers have not yet been changed. Expanding the work may well require some more thought on the driver API within the kernel but, for the most part, the changes are not huge. 

WiFi is, Täht said, the only wireless technology that is fully under our control. We should be taking more advantage of that control to make it work as well as it possibly can; he wishes that there were more developers working in this area. Even a relatively small group has been able to make some significant progress in making WiFi work as it should, though; we will all be the beneficiaries of this work in the coming years. 

[Your editor thanks LWN subscribers for supporting his travel to LPC.]
