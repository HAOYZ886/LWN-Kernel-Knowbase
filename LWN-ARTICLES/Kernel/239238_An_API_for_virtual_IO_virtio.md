---
title: "An API for virtual I/O: virtio"
url: https://lwn.net/Articles/239238/
date: "July 11, 2007"
category: "Virtualization-virtio"
author: 
---

Linux has an abundance of virtualization choices, each with its own way of dealing with I/O. A recent set of kernel patches, submitted to the kernel-virtualization mailing list by Rusty Russell, would allow different virtualization implementations to share drivers by using a virtual I/O interface called _virtio_. There have been several public iterations of the interface with the latest, [draft IV](<http://lwn.net/Articles/240626/>), narrowing in on what appears to be an acceptable solution, at least with the virtualization folks. 

There are always questions about adding yet another layer into the kernel, but the advantages for virtio are numerous. Russell outlines several in one of his [posts](<http://lwn.net/Articles/241104/>) to the kernel-virtualization list. There is some amount of urgency in devising a solution because several of the virtualization projects are either working on or reworking their virtual I/O. If an established mechanism, that already provides working block and network drivers existed, those projects, as well as any newcomers, would be likely to use it.

Another key element is to try and prevent a major proliferation of kernel drivers each handling slightly different virtual block I/O. Trying to tune and maintain those drivers could become a major headache, so virtio separates the guest Linux side of the driver from the code that is specific to the hypervisor implementation. Each group of developers can maintain the code on their side of the API without changing the other, unless, of course, the virtio API itself needs to change. It is likely that _some_ kind of virtual I/O will be adopted, as the kernel developers are likely to be unwilling to merge new drivers for each different virtualization mechanism that comes along; some commonality is required. 

The basic abstraction used by virtio is a "buffer", which consists of a `struct scatterlist` array. The array contains "out" entries describing data destined for the underlying hypervisor driver, as well as "in" entries for that driver to store data to return to the guest driver. The order is fixed (out followed by in) and a count of each is part of the buffer description, which allows the hypervisor driver to determine what it has. This buffer abstraction encapsulates everything needed to communicate data to be written to or read from the hypervisor driver and, eventually, the underlying device. A guest driver, that uses the virtio interface, hands off buffers to the hypervisor driver and awaits their completion. 

At its core, the virtio API is a set of functions that are provided by the hypervisor driver to be used by the guest: 

```
struct virtqueue_ops {
            int (*add_buf)(struct virtqueue *vq,
                           struct scatterlist sg[],
                           unsigned int out_num,
                           unsigned int in_num,
                           void *data);
    
            void (*sync)(struct virtqueue *vq);
    
            void *(*get_buf)(struct virtqueue *vq, unsigned int *len);
    
            int (*detach_buf)(struct virtqueue *vq, void *data);
    
            bool (*restart)(struct virtqueue *vq);
        };
```

This operations vector is initialized by the hypervisor and passed to the guest driver using a `probe()` function. The guest then sets up its data structures and registers with its kernel as a block or network device driver. 

The basic operation uses `add_buf()` to register one or more buffers with the hypervisor driver. That driver is kicked via the `sync()` call to start processing the buffers. Each `struct virtqueue` has a callback associated with it which will be called when some buffers have completed. The guest then calls the `get_buf()` function to retrieve completed buffers. To support polling, which is used by network drivers, `get_buf()` can be called at any time, returning NULL if none have completed. The guest driver can disable further callbacks, at any time, by returning zero from the callback. The `restart()` routine is then used to re-enable them. Finally, the `detach_buf()` call is used during shutdown to cancel the operation indicated by the buffer and to retrieve it from the hypervisor driver. 

As part of his patches, Russell has working example block and network drivers using the virtio interface. Each uses the virtio API differently, and the requirements of each kind of device has pushed the evolution of the interface into its current form. He has also posted an example of a driver implementing virtio for his [lguest](<http://lguest.ozlabs.org/>) hypervisor. 

The block driver uses a protocol that the buffer always has at least one out and in element. The first element passes the sector and type (read or write) information to the hypervisor driver and the first in element receives the status of the request. For a write, there are additional out elements, whereas for a read, there are additional in elements. When the I/O completes, the callback is invoked and the `get_buf()` calls return the completed buffers. 

The network driver uses separate virtqueues for sending and receiving packets which allows it to avoid any locking between the two. Each side only uses half of the scatterlist, out for sending and in for receiving. One of the major differences from "draft III" is combining the two types of buffers; previously there were "inbufs" and "outbufs" and the operations vector had calls for each type. By noticing that they could be combined while still supporting single direction buffers, Russell has halved the number of operations that need to be implemented. 

Currently, a hypervisor that wants to provide virtio devices to its guests must arrange to call the `virtblock_probe()` or `virtnet_probe()` functions. Any device discovery must be handled by the hypervisor and the guest driver is linked to the hypervisor driver at compile time. Dynamic, mix and match, hypervisor/guest combinations are not yet available, but will be down the road; proposals are already being floated on the kernel-virtualization list. 

In a blog [posting](<http://ozlabs.org/~rusty/index.cgi/tech/2007-05-21.html>), Russell describes the tension between performance and abstraction: 

The danger is to come up with an abstraction so far removed from what's actually happening that performance sucks, there's more glue code than actual driver code and there are seemingly arbitrary correctness requirements. But being efficient for both network and block devices is also quite a trick. 

It remains to be seen if the performance can live up to the needs of the various virtualization projects. If it does, and the interface is abstract enough to handle the kinds of virtual devices required, we should see some kind of push to get it included in the kernel sometime soon.
