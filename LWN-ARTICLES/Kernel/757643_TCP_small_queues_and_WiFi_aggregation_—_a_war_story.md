---
title: TCP small queues and WiFi aggregation — a war story
url: https://lwn.net/Articles/757643/
date: "June 18, 2018"
category: "Networking-Wireless"
author: "June 18, 2018 Contributed by Carlo Grazia and Natale Patriciello"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

June 18, 2018

Contributed by Carlo Grazia and Natale Patriciello

This article describes our findings that connected [TCP small queues](<https://lwn.net/Articles/507065/>) (TSQ) with the behavior of advanced WiFi protocols and, in the process, solved a throughput regression. The resulting patch is already in the mainline tree, so before continuing, please make sure your kernel is updated. Beyond the fix, it is delightful to travel through history to see how we discovered the problem, how it was tackled, and how it was patched. 

The academic life is full of rewards; one of ours was the moment in which three USB/WiFi 802.11ab/g/n dongles arrived. We bought dongles with an Atheros chipset because both the software driver and the firmware are available and modifiable. We were using the `ath9k_htc` kernel module with the default configuration. We compiled the latest (at the time) available kernel (4.13.8), and then we started the access point to create an 802.11n network to build the core of our future testbed for vehicular communications. 

We started some simple tests with `ping` and `iperf` to check the connectivity, the distribution of IP addresses, and our custom DNS, which translates the names of our services into IP addresses. The nominal transfer rate of the dongles is 150Mb/s, but what we saw on the screen was disappointing: an upload `iperf` connection, no matter which options were used, was able to reach only 40Mb/s. Using another operating system as a client, we were able to achieve 90Mb/s, leaving out a problem with the server. Even with the newer kernel release (4.14), we did not see anything in the kernel messages that would have been correlated with a hardware or a driver failure. To stress-test the equipment, we started a UDP transmission at a ludicrous speed. Not so surprisingly, we arrived almost at 100Mb/s. It was clear that the root of the problem was in the TCP module or its interactions with the queueing disciplines, so the journey began. 

The next step involved the `tc` command. We started listing the default queueing discipline and modifying its parameters. By default, we were using the `mq` queuing discipline, which instantiated an [FQ-Codel](<https://tools.ietf.org/html/rfc8290>) queuing discipline for each outgoing hardware queue. With another driver, such as `ath9k`, the entire queuing layer is bypassed and a custom version of it, without the possibility of tuning or modifying the queueing discipline, is deployed inside the kernel WiFi subsystem. With `ath9k_htc` driver, instead, we still had the chance to play with the queuing discipline type and parameters. We opted for using the most basic (but reliable) discipline, `pfifo_fast`. But nothing changed. 

We were using the default [CUBIC](<https://en.wikipedia.org/wiki/CUBIC_TCP>) congestion-control module. Despite the recent hype around [BBR](<https://lwn.net/Articles/701165/>), we decided to stick with CUBIC because it has always just worked and never betrayed us (until now, as it seems). Just for a try, we switched to BBR, but things got worse than before; the throughput dropped by 50%, never passing the 20Mb/s line. To do all the tests, we employed [Flent](<https://flent.org/>), which also gives latency results. All the latencies were low; we never exceeded a couple of milliseconds of delay. In our experience, a low throughput with a low latency indicates a well-known problem: starvation. So the question became: what was limiting the number of segments transmitted by the client? 

In 2012, with [commit 46d3ceabd8d9](<https://git.kernel.org/linus/46d3ceabd8d9>), TCP small queues were introduced. Their initial objective was to prevent TCP sockets from queuing more than 128KB of data in the network stack. In 2013, the algorithm was updated to have a dynamic limit. Instead of the fixed value, the limit was defined as either two segment's worth of data or an amount of data that corresponds to a transmission time of 1ms at the current (guessed) transmission rate. The calculation of the transmission rate was added some months earlier, with the objective of calculating the proper sizing of segments when TCP segmentation offload is in use, along with the introduction of a packet scheduler (FQ) able to spread out the sent segments over an interval. 

However, the first reports suggested that the amount of data queued was too low in some subsystems, such as WiFi. The reason behind this was the impossibility, for the WiFi driver, of performing [frame aggregation](<https://en.wikipedia.org/wiki/Frame_aggregation>), due to the lack of data in the driver's queue. The aggregation technique combines multiple packets into a single frame to reduce the constant overhead for over-the-air transmission. Preventing aggregation is a sure way to wreck throughput. 

In response, a minimum amount of buffering (128KB) was restored in [commit 98e09386c0ef4](<https://git.kernel.org/linus/98e09386c0ef4>). One year later, a [refactoring patch](<https://git.kernel.org/linus/605ad7f184b60>) for segmentation offload sizing introduced a small modification that, as we will see, changed the situation dramatically. The 128KB value was changed from being a lower bound to an upper bound. If the amount of data queued was forced to be less than 128KB, what would happen to the WiFi aggregation? 

Fast forwarding to the 4.14 kernel, we started to think how to tune these thresholds. First of all, the function that decides (even in recent kernels) how much TCP data is allowed to enter the network stack is [`tcp_small_queue_check()`](<https://elixir.bootlin.com/linux/latest/source/net/ipv4/tcp_output.c#L2191>): 

```
static bool tcp_small_queue_check(struct sock *sk, const struct sk_buff *skb,
        	   			      unsigned int factor)
        {
    	unsigned int limit;
    
    	limit = max(2 * skb->truesize, sk->sk_pacing_rate >> 10);
    	limit = min_t(u32, limit, sock_net(sk)->ipv4.sysctl_tcp_limit_output_bytes);
    	limit <<= factor;
    
    	/* ... */
        }
```

The `limit` is calculated as the maximum of two full-size segments and ~1ms of data at the current rate. Then the minimum of this value and the 128KB threshold is used (to be in sync with the kernel history, we must say the default value was raised to 256KB in 2015). We started to wonder what would happen if we patched out the possibility of setting a lower bound on the amount of data that could be enqueued. We then modified in the most obvious way the above function to get the following results: 

> ![\[Throughput results\]](https://static.lwn.net/images/2018/tsq-limit.png)

The first column represents the results using the pre-fix parameters for TSQ (two segments or ~1ms of data at the current rate). In the second, we forced at least 64KB to be queued. As we can see, the throughput increased by 20Mb/s, but also the delay (even if the latency increase is not as pronounced as the throughput increase). Then we tested the original configuration in which the lower bound was 128KB; the throughput exceeds the 90Mb/s value, with an added latency of 2ms. It is enough to have 128KB of data queued to have the proper aggregation behavior, at least with our hardware. Increasing that value (we plotted up to 10MB) does not improve in any way the throughput, but it worsens the delay. Even the case in which the TSQ is entirely disabled did not add any improvement to the situation. We found the cause of the problem: a minimum value of data should be enqueued to ensure that frame aggregation works. 

After the testing phase, we realized that putting back fixed byte values would be the wrong choice because, for slow flows, we would only have increased the latency. But, thanks to the modifications done to support BBR, we do know what the flow's current rate is: why not use it? In fact, in [commit 3a9b76fd0db9f](<https://git.kernel.org/linus/3a9b76fd0db9f>), pushed at the end of 2017, the logic of TSQ was enriched by the possibility, for a device driver, to increase the number of milliseconds worth of data that can be queued. The best value for throughput that worked in all the hardware we tested was in between 4-8ms at the flow rate. So, we shared our results, and some weeks later a patch was accepted. In your latest kernel, thanks to [commit 36148c2bbfbe](<https://git.kernel.org/linus/36148c2bbfbe>), your WiFi driver can allow TCP to queue enough data to solve the aggregation problem with a negligible impact on latency. 

The networking stack is complicated (what is simple in kernel space?). For sure, it is not an opaque black box, but instead, it is an orchestrated set of different pieces of knowledge, reflected into layers that can, sometimes, make incompatible choices. As a lesson, we learned that the relationship between latency and throughput on different technologies is not the same, and aggregation in wireless technologies is more common than we initially thought. Moreover, as a community, we should start thinking about automated tests that can give an idea of the performance impact of a patch under different technologies and in a wide range of contexts, from the 40Gb/s device of a burdened server to the 802.11ab/g/n USB dongle connected to a Raspberry Pi. 

[The authors would like to thank Toke Høiland-Jørgensen for his support and the time he dedicated to the Flent tool, to the WiFi drivers, and to gather the results from the ath9k and ath10k drivers.]
