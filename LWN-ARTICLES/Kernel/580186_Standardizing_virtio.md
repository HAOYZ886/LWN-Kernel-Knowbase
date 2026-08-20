---
title: Standardizing virtio
url: https://lwn.net/Articles/580186/
date: "January 15, 2014"
category: "Virtualization-virtio"
author: "By Jonathan Corbet January 15, 2014 linux.conf.au 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 15, 2014

* * *

[linux.conf.au 2014](<https://lwn.net/Archives/ConferenceByYear/#2014-linux.conf.au>)

The Linux kernel has seen the development of a wide range of APIs over the years, but few of those have been further developed into an official standard blessed by a recognized standards body. The [virtio](<http://wiki.libvirt.org/page/Virtio>) mechanism, which facilitates the implementation of virtual devices in guest systems running under hypervisors like KVM, may soon be an exception. Rusty Russell is the chief developer behind that effort; he started his [2014 linux.conf.au](<http://linux.conf.au/>) talk by noting that it is still true that one can't fill a lecture hall by talking about standards; indeed, there were one or two empty seats in the room to back up that claim. 

#### "What are the IP issues?"

I/O to virtual devices, Rusty said, differs from real device I/O in a few significant ways. With bare-metal devices, access to device registers tends to be quite fast, but I/O register access for virtual devices, which must be mediated by the hypervisor, is rather slower. On the other hand, access to memory from virtual devices is direct and fast, while real devices require an expensive DMA setup operation. These differences drive people to create paravirtualized drivers (drivers that are aware that they are dealing with virtualized devices) in order to get the best performance. Creating a special class of devices for virtualized guests is horrible, he said, but if you're going to do something that's really horrible, you should try to do it well. Virtio is thus an attempt to do paravirtualized I/O well. 

A fair amount has happened since virtio got its start with the first implementation in the Linux kernel in 2007\. By 2009, a draft specification existed and, in a development that took Rusty by surprise, Virtualbox 3.1 shipped with virtio-net support. By 2011, Linux had support for the virtio memory-mapped I/O bus. In 2012, the Galaxy Nexus handset used virtio to offload multimedia tasks to hardware [![\[Rusty Russell\]](https://static.lwn.net/images/conf/2014/lca/RustyRussell-sm.jpg)](<https://lwn.net/Articles/580191/>) accelerators; this development, Rusty said, was "cool and random." Adoption is picking up in a number of areas; by later this year, FreeBSD should have support in its [BHyVe](<https://wiki.freebsd.org/action/show/bhyve?action=show&redirect=BHyVe>) hypervisor. 

In 2012, ARM Ltd. decided that it wanted to use virtio in the implementation of its [Fast Models](<http://www.arm.com/products/tools/models/fast-models/>) system. So they contacted Rusty, asking what the "intellectual property issues" were around the virtio specification. He answered that it was all just a blog posting, and that they could do with it as they would; this was evidently not an answer that made ARM's lawyers happy; they contacted lawyers within IBM and the question eventually reached him from the other side. 

There is, Rusty said, a process for publishing a white paper from within IBM. He's not quite sure what that process is, but it was made it clear to him "in a series of long meetings" that it cannot be described as "post the specification on your blog, promote it for years, then wait for somebody to ask about the IP issues." IBM's internal processes, it seems, work a bit differently than that. 

This episode suggested that it was time to put together a proper standard for virtio. At this point, the barriers to adoption of virtio were not technical; instead, they were legal and political. Having a published standard will encourage adoption for larger enterprises which, in turn, will make it harder for other projects to go off and do their own thing. Going through the standardization process also presents an opportunity to fix up a number of small issues that have come up over time. The end goal of the process is to try to create a straightforward, efficient, and extensible standard. 

"Straightforward" implies that, to the greatest extent possible, devices should use existing bus interfaces. Virtio devices see something that looks like a standard PCI bus, for example; there is to be no "boutique hypervisor bus" for drivers to deal with. "Efficient" means that batching of operations is both possible and encouraged; interrupt suppression is supported, as is notification suppression on the device side. "Extensible" is handled with feature bits on both the device and driver sides with a negotiation phase at device setup time; this mechanism, Rusty said, has worked well so far. And the standard defines a common ring buffer and descriptor mechanism (a "virtqueue") that is used by all devices; the same devices can work transparently over different transports. 

#### Changes for virtio 1.0

Another way of putting it was that the standardization effort was undertaken with the goals of keeping the good parts of virtio, discarding the bad parts, and making the ugly parts optional. The first step in that direction was to recast the specification into RFC-style language. Rather than suggesting that a driver "should check" that a given feature is supported before trying to use it, the standard says that drivers "MUST check." And so on. 

One of the first thing authors of virtio drivers will notice is the addition of a new feature bit called `VIRTIO_F_VERSION_1`. It is, he said, the first mandatory feature bit in the standard; it indicates that the driver implements version 1.0 and does not require legacy support. A couple of other feature bits (`F_ANY_LAYOUT` and `F_NOTIFY_ON_EMPTY`) have been removed. The former was the "I actually read the damn standard" bit, Rusty said, while the latter indicated the presence of a bug workaround that was never used, since simply fixing the bug turned out to be a better course of action. 

The in-memory virtqueue layout has been made more flexible; the original version could require large, physically contiguous allocations that may fail on a system with fragmented memory, while version 1.0 splits that allocation up. Virtqueue size can also be negotiated by drivers now. A complex interaction between "multipart descriptors" (arrays of memory descriptors stored outside of the main ring) and the "next" bit (used to create multipart descriptors within the main ring) has simply been removed; nobody was using it anyway, Rusty said. 

The status byte provided by drivers was subject to race conditions, since there was no way to know when the driver had finished accepting (or rejecting) proposed features. There is now a `FEATURES_OK` bit to mark the end of the negotiation process; clearing this bit is also a way of indicating that negotiation has failed. There is a new atomicity counter associated with the optional device-specific configuration area; by checking the counter before and after reading a field in this area, code can notice if something changes and retry accordingly. 

There have been relatively few changes to virtio-net; the biggest is the removal of ~~support~~ the `VIRTIO_NET_F_GSO` bit for generic segmentation offloading (GSO). Supporting GSO was complicated, eventually requiring a few separate feature bits, and the overall feature bit was never used. The virtio-block driver has seen the removal of a number of feature bits; the "barrier" feature was unused, while "flush" is now compulsory. More complicated drivers that used to be implemented with virtio-block, Rusty said, should now use virtio-scsi instead. 

The virtio-balloon driver has a number of problems, including its own approach to endianness issues. It uses unaligned fields for the stats virtqueue, and has a "compulsory optional" feature bit to tell the hypervisor that pages are being pulled out of the balloon. Rather than try to fix these problems, the standard committee chose to simply remove virtio-balloon from the standard altogether. 

Endianness has, Rusty said, been a problem for virtio in general. The initial specification said that byte ordering would be whatever the guest expected; the idea is simple, but it turned out not to be straightforward to implement. The balloon driver got it completely wrong, but it was not the only driver with problems. So, with version 1.0 of the specification, the ordering is simply set to be little-endian. This change will create some difficulties for people working on s390; Rusty thanked them for "taking the bullet" to enable this simplification of the standard. 

The process of creating and publishing the virtio standard is being run through [OASIS](<https://www.oasis-open.org/>), (Organization for the Advancement of Structured Information Standards). Rusty said that he put some time into picking the right organization, looking for one that was interested in the creation of useful standards without a lot of unnecessary hoops to jump through. He was warned during the selection process that some standards groups exist primarily to slow things down, which wasn't what he was after. Thus far, development of the standard through OASIS has been going well. 

The first draft of the standard was released on December 24; Rusty allowed as to how some members of the audience might not have noticed it at the time. The second draft is to be expected "in a few months." The work can all be found on [the OASIS virtio committee page](<http://www.oasis-open.org/committees/virtio>); comments are welcome. The whole process, Rusty said, has taken rather longer than he had hoped and has not always been fun, but the result, with luck, will be a standard for paravirtualized devices that will be widely adopted. 

[Your editor would like to thank linux.conf.au for funding his travel to Perth].
