---
title: The Xen patches
url: https://lwn.net/Articles/183105/
date: "May 10, 2006"
category: "Virtualization-Xen; Xen"
author: 
---

The Xen hypervisor has been the source of large amounts of hype for some time now. The Xen paravirtualization scheme allows the running of guest operating systems, but the guest kernel must be ported explicitly to the "architecture" supported by the hypervisor. Paravirtualization provides strong isolation of virtual machines and can be quite fast, but it cannot run unmodified operating systems on its virtual machines. Many had expected support for Xen to be merged into the mainline by now, but that has not happened. In fact, it is only recently that the Xen patches have even been posted for developer review. A [new set of Xen patches](<http://lwn.net/Articles/182911/>) was posted on May 9, however, giving some insights into how Xen will affect the kernel. 

The patches in the 35-part set fall into two broad categories. The first of those creates a new architecture (a subarchitecture of i386) and a port of the Linux kernel for that architecture. This is the code which is built into the modified kernel which can run as a Xen guest. Some of the more significant changes include: 

  * Allowing for more interrupt vectors. Xen uses pseudo-interrupts for various types of communications with guests, so there needs to be room for more interrupt handlers. 

  * An events mechanism has been built on top of the interrupt management code so that the hypervisor can pass information into guest systems. The virtual machines can also use event channels to communicate with each other. 

  * Much of the i386 initialization code is split out so that subarchitectures can override it. Since a Xen-hosted kernel is not booting on cold hardware, and it will not use a number of hardware features, it will have to initialize itself differently than the host system does. 

  * A version of the [dynamic tick patch](<http://lwn.net/Articles/138969/>) is used to keep idle virtual machines from wasting time servicing timer interrupts. There is also a separate timekeeping implementation which allows guest systems to perform their own timekeeping without having to involve the hypervisor. 

  * A whole range of virtual devices has been provided. These include a console, virtual network interfaces, and virtual block devices. 

Then, there are a couple of changes to the core (host) kernel: 

  * A new set of synchronous bit operations, with names like `synch_set_bit()`. These operations differ from the regular bit operations in that they are always atomic. The regular bit operations will, when built for a uniprocessor system, use less-expensive, non-atomic operations. But that will not work well if a uniprocessor Xen guest runs on an SMP host. 

  * The function `apply_to_page_range()` will call a given function for every page table entry in a given range. This patch seems worth merging ahead of the rest of Xen; currently, code iterating through PTEs duplicates a complicated set of functions for walking through the page table structure. 

There has been a fair amount of comment on the patches, but few objections of great substance. Instead, the Xen developers look to have a long list of nits to address. The most fundamental complaints, perhaps, concern the network driver, which includes its own, built-in ARP implementation. The Xen developers defend this code as being necessary for fast migration of Xen guests. If the ARP code were moved to a more appropriate place - user space, for example - a migration which happens in milliseconds could turn into a one-second (or longer) affair, and that is not a cost the Xen folks want to pay. The addition of files to `/proc` is also unpopular, but that code was already on the list of things to fix. 

When Xen might actually merge is still unclear. There is work to be done still, and it is a large body of code for the developers to work through. But that date is getting closer, now that there is code to discuss.
