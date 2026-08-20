---
title: RapidIO support for Linux
url: https://lwn.net/Articles/139118/
date: "June 8, 2005"
category: Device drivers; RapidIO
author: 
---

One of the patch sets which showed up in the 2.6.12-rc6-mm1 kernel is the [RapidIO subsystem](<https://lwn.net/Articles/138887/>), contributed by Matt Porter (of Montavista). Your editor, being ignorant of the [RapidIO](<http://rapidio.org/>) standard, decided to have a look. RapidIO turns out to be a sort of backplane interconnect intended mainly for embedded systems. It allows for multiple hosts to exist on the same bus and work collaboratively with the available peripherals. It is a sort of highly local area network. 

The RapidIO site provides no end of highly detailed specifications for the truly curious. The rest of us, however, can learn a lot by looking at [a network driver](<https://lwn.net/Articles/138892/>) packaged with the rest of the Linux RapidIO patch. This driver provides a simple example of how to use the API provided by the RapidIO layer; it enables network packets to be exchanged with another host on the RapidIO bus. 

The RapidIO subsystem is integrated with the device model, so it provides the expected structures: `rio_dev` and `rio_driver`. Drivers can register a `probe()` function which enables them to take responsibility for devices (which can be other hosts) as they turn up on the interconnect. The example network driver uses a wildcard ID table so that it is given the opportunity to work with all other devices out there; it will happily send packets to any suitably capable device. 

"Suitably capable," in this case, means that the device implements the two basic primitives used to communicate across the RapidIO interconnect. "Doorbells" are a way of sending simple, out-of-band signals to remote nodes; the doorbells used by the network driver are those which announce device addition and removal events. Most work, however, is done with "mailboxes," essentially a reliable packet delivery service. If one RapidIO device sends a message to another via a mailbox, the lower levels will do their best to ensure that the message arrives uncorrupted and in the right order. 

So how does one RapidIO network node send a packet to another? Taking out the usual overhead and error handling, it comes down to the following: 

```
static int rionet_start_xmit(struct sk_buff *skb, struct net_device *ndev)
        {
            struct rionet_private *rnet = ndev->priv;
    
    	rio_add_outb_message(rnet->mport, rdev, 0, skb->data, skb->len);
        }
```

`rdev` is a `rio_dev` structure corresponding to the destination host on the RapidIO backplane. This call sends the data in the network packet (`skb`) out through the given mailbox to the desired device. When the transmission is complete, the driver will receive a callback so that it can perform any necessary cleanup (freeing the `skb` in this case). 

Packet reception requires setting up a ring of receive buffers, much like one would see in any network driver. In this case, the necessary code looks like: 

```
do {
    	rnet->rx_skb[i] = dev_alloc_skb(RIO_MAX_MSG_SIZE);
    
    	if (!rnet->rx_skb[i])
    	    break;
    
    	rio_add_inb_buffer(rnet->mport, RIONET_MAILBOX,
    			   rnet->rx_skb[i]->data);
        } while ((i = (i + 1) % RIONET_RX_RING_SIZE) != end);
```

The RapidIO subsystem maintains a list of buffers waiting for incoming mailbox messages; new buffers are added with `rio_add_inb_buffer()`. When a message actually shows up, the driver gets a callback (established when the mailbox is allocated), which, in the end, does the following: 

```
if (!(data = rio_get_inb_message(rnet->mport, RIONET_MAILBOX)))
    	break;
        rnet->rx_skb[i]->data = data;
        skb_put(rnet->rx_skb[i], RIO_MAX_MSG_SIZE);
        error = netif_rx(rnet->rx_skb[i]);
```

The code assumes that anything arriving on the given mailbox will be a network packet. Beyond that, little checking is required; all of the details, including data integrity checks, will have been taken care of by the lower levels. 

The list of RapidIO-capable devices is small at the moment, but appears to be growing. As these devices become available, Linux will have the low-level infrastructure needed to support them. The embedded Linux community has often been accused of keeping its work to itself and not contributing back to the kernel as a whole. The contribution of the RapidIO subsystem is another sign that this situation may be changing; that, perhaps, is more welcome than the code itself.
