---
title: Reconsidering network channels
url: https://lwn.net/Articles/192767/
date: "July 25, 2006"
category: "Networking-Channels"
author: 
---

When Van Jacobson [presented his network channels idea](<http://lwn.net/Articles/169961/>) at linux.conf.au last January, he set a bit of a fire in the Linux networking community. By making some significant changes to the processing path for incoming packets, and by pushing most of the work as close as possible to the destination application, Van was able to achieve significant performance improvements - eliminating as much as 80% of the processing overhead on multiprocessor systems. With numbers like that, it seemed like the question of whether Linux would incorporate channels need not even be asked. 

Since then, however, reality has begun to make itself felt - something which reality is wont to do, sooner or later. Which is why David Miller's [latest pronouncement](<https://lwn.net/Articles/192769/>) on network channels reads like this: 

Don't get too excited about VJ netchannels, more and more roadblocks to their practicality are being found every day.... All the original costs of route, netfilter, TCP socket lookup all reappear as we make VJ netchannels fit all the rules of real practical systems, eliminating their gains entirely. 

The issue at hand had to do with the integration of channels and netfilter. The hope had been that packets could be identified and sorted into their respective channels before the netfilter (firewall) processing was done. Then said processing could be performed close to the application, on the same processor. It turns out, however, that netfilter can change the real destination of the packet. So packets must be filtered before entering a channel, and much of the performance benefit of using a channel is lost. 

Alexey Kuznetsov has posted [a detailed criticism of channels](<https://lwn.net/Articles/192772/>), asserting that most of the claimed benefits are illusory. Says Alexey: 

It is an amazing toy. But I see nothing, which could promote its status to practical. Exokernels used to do this thing for ages, and all the performance gains are compensated by overcomplicated classification engine, which has to remain in kernel and essentially to do the same work which routing/firewalling/socket hash tables do. 

Finally, it seems that many of the benefits of channels can be had by carefully taking advantage of the capabilities of modern hardware. In particular, an increasing number of devices can perform simple packet classification and (via targeted interrupts) direct packets to the CPU where the destination application is running. That technique will get rid of the cache misses caused by performing interrupt processing on one processor and protocol processing on another. 

In the end, it appears that yet another seemingly bright scheme may not make the transition into real-world deployments. Some of its core concepts, such as using cache-friendly data structures and trying (even harder) to improve cache locality, will likely influence the future direction of the network stack, however. So, while there may not be a revolutionary new mechanism in the network stack's future, some of the promised performance improvements should eventually be realized anyway. And, [as David says](<https://lwn.net/Articles/192774/>), ""At least, there is less code to write.""
