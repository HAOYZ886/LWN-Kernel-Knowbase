---
title: Network transmit queue limits
url: https://lwn.net/Articles/454390/
date: "August 9, 2011"
category: Networking
author: "By Jonathan Corbet August 9, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 9, 2011

Network performance depends heavily on buffering at almost every point in a packet's path. If the system wants to get full performance out of an interface, it must ensure that the next packet is ready to go as soon as the device is ready for it. But, as the developers working on bufferbloat have confirmed, excessive buffering can lead to problems of its own. One of the most annoying of those problems is latency; if an outgoing packet is placed at the end of a very long queue, it will not be going anywhere for a while. A classic example can be reproduced on almost any home network: start a large outbound file copy operation and listen to the loud complaints from the World of Warcraft player in the next room; it should be noted that not all parents see this behavior as a bad thing. But, in general, latency caused by excessive buffering is indeed worth fixing. 

One assumes that the number of Warcraft players on the Google campus is relatively small, but Google worries about latency anyway. Anything that slows down response makes Google's services slower and less attractive. So it is not surprising that we have seen various latency-reducing changes from Google, including the [increase in the initial congestion window](<https://lwn.net/Articles/427104/>) merged for 2.6.38. A [more recent patch](<https://lwn.net/Articles/454378/>) from Google's Tom Herbert attacks latency caused by excessive buffering, but its future in its current form is uncertain. 

An outgoing packet may pass through several layers of buffering before it hits the wire for even the first hop. There may be queues within the originating application, in the network protocol code, in the traffic control policy layers, in the device driver, and in the device itself - and probably in several other places as well. A full solution to the buffering problem will likely require addressing all of these issues, but each layer will have its own concerns and will be a unique problem to solve. Tom's patch is aimed at the last step in the system - buffering within the device's internal transmit queue. 

Any worthwhile network interface will support a ring of descriptors describing packets which are waiting to be transmitted. If the interface is busy, there should always be some packets buffered there; once the transmission of one packet is complete, the interface should be able to begin the next one without waiting for the kernel to respond. It makes little sense, though, to buffer more packets in the device than is necessary to keep the transmitter busy; anything more than that will just add latency. Thus far, little thought has gone into how big that buffer should be; the default is often too large. On your editor's system, `ethtool` says that the length of the transmit ring is 256 packets; on a 1G Ethernet, with 1500-byte packets, that ring would take almost 4ms to transmit completely. 4ms is a fair amount of latency to add to a local transmission, and it's only one of several possible sources of latency. It may well make sense to make that buffer smaller. 

The problem, of course, is that the ideal buffer size varies considerably from one system - and one workload - to the next. A lightly-loaded system sending large packets can get by with a small number of buffered packets. If the system is heavily loaded, more time may pass before the transmit queue can be refilled, so that queue should be larger. If the packets being transmitted are small, it will be necessary to buffer more of them. A few moments spent thinking about the problem will make it clear that (1) the number of packets is the wrong parameter to use for the size of the queue, and (2) the queue length must be a dynamic parameter that responds to the current load on the system. Expecting system administrators to tweak transmit queue lengths manually seems like a losing strategy. 

Tom's patch adds a new "dynamic queue limits" (DQL) library that is meant to be a general-purpose queue length controller; on top of that he builds the "byte queue limits" mechanism used within the networking layer. One of the key observations is that the limit should be expressed in bytes rather than packets, since the number of queued bytes more accurately approximates the time required to empty the queue. To use this code, drivers must, when queueing packets to the interface, make a call to one of: 

```
void netdev_sent_queue(struct net_device *dev, unsigned int pkts, unsigned int bytes);
        void netdev_tx_sent_queue(struct netdev_queue *dev_queue, unsigned int pkts,
    			      unsigned int bytes);
```

Either of these functions will note that the given number of `bytes` have been queued to the given device. If the underlying DQL code determines that the queue is long enough after adding these bytes, it will tell the upper layers to pass no more data to the device for now. 

When a transmission completes, the driver should call one of: 

```
void netdev_completed_queue(struct net_device *dev, unsigned pkts, unsigned bytes);
        void netdev_tx_completed_queue(struct netdev_queue *dev_queue, unsigned pkts,
    				   unsigned bytes);
```

The DQL library will respond by reenabling the flow of packets into the driver if the length of the queue has fallen far enough. 

In the completion routine, the DQL code also occasionally tries to adjust the queue length for optimal performance. If the queue becomes empty while transmission has been turned off in the networking code, the queue is clearly too short - there was not time to get more packets into the stream before the transmitter came up dry. On the other hand, if the queue length never goes below a given number of bytes, the maximum length can probably be reduced by up to that many bytes. Over time, it is hoped that this algorithm will settle on a reasonable length and that it will be able to respond if the situation changes and a different length is called for. 

The idea behind this patch makes sense, so nobody spoke out against it. Stephen Hemminger did express concerns about the need to add explicit calls to drivers to make it all work, though. The API for network drivers is already complex; he would like to avoid making it more so if possible. Stephen thinks that it should be possible to watch traffic flowing through the device at the higher levels and control the queue length without any knowledge or cooperation from the driver at all; Tom is not yet convinced that this will work. It will probably take some time to figure out what the best solution is, and the code could end up changing significantly before we see dynamic transmit queue length control get into the mainline.
