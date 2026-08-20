---
title: Linux and TCP offload engines
url: https://lwn.net/Articles/148697/
date: "August 22, 2005"
category: Networking; TCP
author: 
---

The TCP/IP protocol suite takes a certain amount of CPU power to implement. So it is not surprising that network adapter manufacturers have long been adding protocol support to their cards. This support can vary from the simple (checksumming of packets, for example) through to full TCP/IP implementations. An adapter with full protocol support is often called a TCP offload engine or TOE. 

Linux has never supported the TOE features of any network cards. For some time, there had not even been much discussion of TOE support. The topic has returned, however, with [this patch adding TOE support](<https://lwn.net/Articles/147289/>) which was posted by Scott Bardone of Chelsio Communications. This TOE patch is clearly intended to support Chelsio's line of network adapters, but it has been coded as a more generic "open TOE" framework. The Chelsio folks would very much like to see this patch merged for the 2.6.14 kernel release. 

Those who are curious about the TOE patch can go in and look at the code; it is relatively straightforward. At its core, it creates a new type of extended network device (`struct toedev`) with an additional set of methods: 

```
int (*open)(struct toedev *dev);
    	int (*close)(struct toedev *dev);
    	int (*can_offload)(struct toedev *dev, struct sock *sk);
    	int (*connect)(struct toedev *dev, struct sock *sk);
    	int (*send)(struct toedev *dev, struct sk_buff *skb);
    	int (*recv)(struct toedev *dev, struct sk_buff **skb, int n);
    	int (*ctl)(struct toedev *dev, unsigned int req, void *data);
    	void (*neigh_update)(struct net_device *lldev,
    			     struct toedev *dev,
    			     struct neighbour *neigh, int fl);
```

There are various hooks sprinkled through the TCP code to detect when a TOE-capable device is being used and call the appropriate method rather than performing the TCP processing in the kernel. One assumes that the patch works as advertised, but its chances of getting into the kernel appear to be relatively small. There is a very long list of objections which have been raised, including: 

  * The TOE code must, by necessity, hook deeply into the Linux TCP implementation. These hooks will make it harder to make high-level TCP changes in the future. The TOE patch thus represents a long-term maintenance burden. 

  * TOE shorts out much of the Linux networking code. In the process, it cuts out little features like netfilter, traffic control, and more. So a Linux system using TOE will lack many of the capabilities which characterize the Linux networking stack. The networking hackers can already foresee the interminable series of "why doesn't my TOE adapter support netfilter?" questions which will go their way. 

  * The Linux networking stack is easy to fix when a bug or security issue comes up. If a security problem turns up in a TOE adapter, instead, there is very little which can be done to fix it. 

  * The performance benefits from TOE are minimal at best. Even if a TOE adapter and software stack currently outperforms "dumber" adapters for very high networking speeds (10G currently, say), that advantage tends to disappear by the time those speeds are in common use. Jeff Garzik [claims](<https://lwn.net/Articles/148700/>) that 100Mb/s TOE adaptors (which used to be the bleeding-edge high speed) are now slower than the Linux networking stack. So any performance advantage from TOE is a temporary thing, but, once it is merged, the code must be supported forever. 

There is also the inconvenient little detail that a company called Alacritech owns [several patents](<http://www.taborcommunications.com/dsstar/04/0817/108758.html>) relating to TOE. It recently [used those patents](<http://www.eet.com/news/latest/showArticle.jhtml?articleID=166401437>) to extract money from Microsoft, which is including TOE support in its upcoming Windows release. This, alone, would almost certainly cause distributors to disable TOE support, even if it were to find its way into the kernel. (For the record, Chelsio [claims](<https://lwn.net/Articles/148790/>) to have done its legal homework, but not everybody finds that claim to be convincing). 

Will it find its way in? Not if David Miller [has anything to say](<https://lwn.net/Articles/148701/>) on the matter: 

I am still very much against TOE going into the Linux networking stack. There are ways to obtain TOE's performance without necessitating stateful support in the cards, everything that's worthwhile can be done with stateless offloads. 

There is essentially zero chance of a networking patch being merged over David's objections, so the TOE developers have an uphill road ahead of them. 

One might well ask: if TOE cannot be merged, how will Linux maintain competitive speeds as networks get faster? A big area of interest, currently, is offloading parts of the protocol which do not require great intelligence or state in the card. The kernel already supports TCP segmentation offloading (TSO), where an adapter can create TCP packets out of a large array of data. TSO reduces the necessary CPU power, bus overhead, and cache impact to send a series of packets, but it still does not require that the adapter actually know anything about specific TCP connections. There is talk of using a similar technique for incoming packets: an adapter could merge a configurable set of incoming packets into a single array, thus reducing the demands on the rest of the system. One way or another, the networking stack is likely to keep up with the demands of current hardware. 

It has often been said that a maintainer's real job is to say "no" to patches. Not all features are worth their (very real) cost, and merging some patches can be detrimental to the kernel in the long run. For years, the networking maintainers have felt that TOE support is the kind of patch which should not be accepted, and the current implementation appears not to have changed their minds. TOE appears to be one of those ideas which never really goes away, however, so chances are good that we will see this debate again in the future.
