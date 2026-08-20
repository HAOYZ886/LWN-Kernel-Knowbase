---
title: The return of network channels
url: https://lwn.net/Articles/260880/
date: "December 4, 2007"
category: "Networking-Channels"
author: "By Jonathan Corbet December 4, 2007"
---

By **Jonathan Corbet**  
December 4, 2007

The network channels concept was [first aired by Van Jacobson](<http://lwn.net/Articles/169961/>) almost two years ago at linux.conf.au 2006. This idea promises much-improved networking performance by pushing processing of network data as close to the end point as possible - perhaps even into user space. By getting the kernel out of the packet processing business and by keeping that processing in a single place (on the same CPU), channel schemes hope to minimize cache misses, context switches, and other performance-degrading activities. Channels have had a rough encounter with the real world, though; when one starts to consider needs like packet filtering, address translation, and so on, it gets hard to maintain the simplicity upon which the performance of channels relies. So, two years later, there is no channels implementation which is even close to merging into the mainline. 

That does not mean that no work is happening in this area, though. Evgeniy Polyakov, perhaps the most discouragement-resistant hacker out there, continues to develop his channel patches; the [22nd release](<http://lwn.net/Articles/260873/>) came out on December 4\. 

This version of the patch has a well-defined internal structure to allow kernel code to hook into channels. The best-developed mode, however, is the one which simply transfers packets to and from user space. To that end, there is a new system call: 

```
int netchannel_control(struct unetchannel_control *ctl);
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The full contents of the `unetchannel_control` structure can be seen in the patch. The more important fields are: 

  * `cmd`, describing the action that the calling process wishes to execute. Unlike previous versions of the patch, the current code only supports one action: `NETCHANNEL_CREATE`, which makes a new channel. 

  * `type`, the type of the channel to create. At the moment, the only implemented type is `NETCHANNEL_COPY_USER`, which copies packets to and from user space. 

  * `unc.data` which describes the channel to be created: it contains source and destination addresses and ports and a protocol number. 

Once a network channel is created, it is added to a search tree which is oriented toward blindingly-fast lookups. There is a new hook in the packet receive code which looks up each incoming packet in that tree; packets which do not turn up a hit there are processed normally by the kernel's networking stack. Any packet whose addresses, ports, and protocol are matched by an entry in the tree, however, is shunted over to the channel code before even being queued by the network stack. 

The final piece (on the receive side) is a simple `read()` implementation. A process wishing to receive a packet from a network channel need only read the associated file descriptor and the next available packet will be copied into the supplied buffer. It would, of course, be nice to do away with that copy operation, but that is a hard trick to carry out: the packet must be received before its destination is known. There are network adapters which can direct packets based on their header information, but the current netfilter does does not have the driver API enhancements which would be required to use that capability for zero-copy packet reception. 

Similarly, a `write()` operation causes the associated packet to be copied into the kernel and fed into the networking stack at a fairly low level. There is currently no zero-copy write support. 

Evgeniy clearly has zero-copy operations in mind, though, probably using his [network allocator patch](<http://lwn.net/Articles/195292/>). Even without that feature, though, the channel code, when used with his [user-space network stack](<http://tservice.net.ru/~s0mbre/old/?section=projects&item=unetstack>) appears to be quite fast. Some posted [benchmark results](<http://tservice.net.ru/~s0mbre/blog/2006/12/21#2006_12_21>) claim significant improvements over the core Linux networking stack - three times the maximum bandwidth with one-third of the CPU usage when small packets are being transferred. For larger (4096-byte) packets the performance improvements essentially disappear - most likely the cost of copying the packets into and out of the kernel is the dominating factor there. 

Improvements in small-packet performance are welcome: there are a number of applications, including high-end financial trading, which require large numbers of small transfers. The addition of zero-copy logic has the potential to make the large-packet performance better as well. The real test, though, will be the addition of all of the other features expected by contemporary networking users, most of which are currently absent from the channels implementation. There are hooks in the code aimed at the insertion of per-packet processing; they could be used for filtering, address translation, traffic control, or any of the other things that one might want to have. Whether those hooks can be used without killing the performance advantages of channels remains to be seen, though. But one suspects that Evgeniy will not give up until he has an answer to that question.
