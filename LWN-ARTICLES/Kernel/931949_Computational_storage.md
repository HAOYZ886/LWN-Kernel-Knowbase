---
title: Computational storage
url: https://lwn.net/Articles/931949/
date: "May 17, 2023"
category: Computational storage; NVMe
author: "By Jake Edge May 17, 2023 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
May 17, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

A new development in the NVMe world was the subject of a combined storage and filesystem session led by Stephen Bates at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). Computational storage namespaces will allow NVMe devices to offer various types of computation—anything from simple compression through complex queries and data manipulations—to be performed on the data stored on the device. 

Bates began the session by noting that he had seen a video of a talk recently where a professor had calculated that about two-thirds of the energy used in "high-scale distributed systems" is spent on data movement. So, large Hadoop clusters or AI-training frameworks are using an enormous amount of energy just to move data around, without actually doing any real work ("not actually changing any bits"). That is done because we generally move our data from storage into memory in order to operate on it, which is sort of inherent in the [von Neumann architecture](<https://en.wikipedia.org/wiki/Von_Neumann_architecture>). The storage systems are typically somewhat removed from the processor and, even if they are attached via, say, the PCIe bus, they still take a lot of power. 

The idea behind computational storage, he said, is to see if parts of the computation can be pushed out to the storage layer. NVMe is more than just a good storage protocol, it is "an awesome protocol for telling something to do something". That request can be sent via a variety of mechanisms (e.g. direct attached, over fabric) and it can all be done efficiently because of the NVMe design. A command is sent as a 64-byte submission queue entry (SQE), which will result in a 16-byte completion queue entry (CQE) at some later point that gives the completion status of the command. NVMe is "super parallel" with large numbers of queues that can be split up among the cores in an SMP system. The queues can be "incredibly deep" and do not need to be processed in order. In addition, those 64-byte SQEs do not have to request a storage operation; "tell me the temperature in Idaho" could be a valid NVMe command, he said. 

An NVMe controller may not just be for a single SSD, but could be some rack-sized storage system with lots of processing power available to it. Two new namespace types are being defined; one is called "subsystem local memory", which is byte-addressable in order to facilitate the computational processing on the local device. It can be populated from a traditional, logical block address (LBA) namespace or from an object-storage namespace; the subsystem local memory becomes the physical address space for the computation. 

The second new namespace is for computational programs, which can perform certain actions on the data in the local memory. Those actions could be vendor-specific, simple things like compression, executing eBPF programs, and more. In theory, you could provide a Docker image to the NVMe device and it could run on the local data, he said. In some measurements that he and his colleagues have done, they have seen order-of-magnitude reductions in energy usage by pushing computations out to the storage devices. 

The challenge is in figuring out how to break up the application's computation to take advantage of this. There are various efforts from Samsung and within the [Storage Networking Industry Association](<https://www.snia.org/>) (SNIA) to define a user-space API for computational storage. There are plans for a library that would let applications discover devices and their computational capabilities; it would provide frameworks for pushing programs in various languages out to the devices. There would also be interfaces for executing the programs and gathering the results. 

Bates said that it is unlikely that support for this goes directly into the kernel. He noted that storage-track organizer Javier González and Samsung have been working on adding infrastructure to io_uring to allow passing arbitrary NVMe commands from user space, though there are some security concerns that need to be worked out. That way the kernel would not need to learn about these new commands; it would simply send the 64-byte messages and return the 16-byte replies. That ability will be useful even beyond the computational-storage realm as it will allow experimenting with new NVMe commands. As new commands get proven, they could then move into the kernel proper. 

When Bates started looking at this problem, he was focused on SSDs with computational storage, but these days he thinks that storage arrays with computational-storage facilities are the more compelling target. He also sees the object-storage namespace as being important for doing computations; the mapping from files to LBAs in filesystems is complicated, so avoiding that completely makes sense. If a filesystem is being used, the library for computational storage would need to be able to turn a file name into a list of LBAs to operate on. González said that some of the work on [ublk](<https://lwn.net/Articles/903855/>) may help with parts of that. 

For storage arrays, the cores controlling the arrays may only be fully utilized when the array is writing at its maximum rate, Bates said. At other times, there may be excess capacity and power availability in an array for doing other kinds of work. Some of his customers have workloads that are highly time-dependent, so there are largely idle periods where computation could be done on the storage array, which is much more efficient than moving the data to host systems for performing the computation.
