---
title: "UKUUG: Arnd Bergmann on interconnecting with PCIe"
url: https://lwn.net/Articles/307482/
date: "November 19, 2008"
category: Device drivers; PCIe
author: "By Jake Edge November 19, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
November 19, 2008

PCI express (PCIe) is not normally considered as a way to connect computers, rather it is a bus for attaching peripherals, but there are advantages to using it as an interconnect. Kernel hacker Arnd Bergmann gave a presentation at the recent [UKUUG Linux 2008 conference](<http://www.ukuug.org/events/linux2008/>) on work he has been doing on using PCIe for IBM. He outlined the current state of Linux support as well as some plans for the future. 

The availability of PCIe endpoints for much of the hardware in use today is one major advantage. By using PCIe, instead of other interconnects such as [InfiniBand](<http://en.wikipedia.org/wiki/Infiniband>), the same throughput can be achieved with lower latency and power consumption. Bergmann noted that avoiding using a separate InfiniBand chip saves 10-30 watts which adds up rather quickly on a 30,000 node supercomputer. 

There are some downsides to PCIe as well. There is no security model, for example, so a root process on one machine can crash other connected machines. There is also a single point of failure because if the PCIe root port goes down, it takes the network with it or, as Bergmann puts it: ""if anything goes wrong, the whole system goes down"". PCIe lacks a standard high-level interface for Linux and there is no generic code shared between the various drivers--at least so far. 

As an example of a system that uses PCIe, Bergmann described the "Roadrunner" supercomputer that is currently the fastest in existence. It is a cluster of hybrid nodes, called "Triblades", each of which has one Opteron blade along with two Cell blades. The nodes are connected with InfiniBand, but PCIe is used to communicate between the processors within each node by using the Opteron root port and PCIe endpoints on the Cells. 

There is other hardware that uses PCIe in this way, including the Fixstars GigaAccel 180 accelerator board and an embedded PowerPC 440/460 system-on-a-chip (SoC) board, both of which use the same Axon PCIe device. Bergmann also talked about PCIe switches and non-transparent bridges that perform the same kinds of functions as networking switches and bridges. Bridges are called "non-transparent" because they have I/O remapping tables--sometimes IOMMUs--that can be addressed by the two root ports that are connected via the bridge. These bridges may also have DMA engines to facilitate data transfer without host processor control. 

Bergmann then moved on to the software side of things, looking at the drivers available--and planned--to support connection via PCIe. The first driver was written by Mercury Computers in 2006 for a Cell accelerator board and is now "abandonware". It has many deficiencies and would take a lot of work to get it into shape for the mainline. 

Another choice is the driver used in the Roadrunner Triblade and the GigaAccel device which is vaguely modeled on InfiniBand. It has an interface that uses custom `ioctl()` commands that implement just eight operations, as opposed to hundreds for InfiniBand. It is ""enormous for a Linux device driver"", weighing in at 13,000 lines of code. 

The Triblade driver is not as portable as it could be, as it is very specific to the Opteron and Cell architectures. On the Cell side, it is implemented as an Open Firmware driver, but the Opteron side is a PCIe driver. There is a lot of virtual ethernet code mixed in as well. Overall, it is not seen as the best way forward to support these kinds of devices in Linux. 

Another approach was taken by a group of students sponsored by IBM who developed a virtual ethernet prototype to talk to an IBM BladeCenter from a workstation by way of a non-transparent bridge. Each side could access memory on the other by using `ioremap()` on one side and `dma_map_single()` on the other. By implementing a virtio driver, they did not have to write an ethernet driver, as the virtio abstraction provided that functionality. The driver was a bit slow, as it didn't use DMA, but it is a start down the road that Bergmann thinks should be taken. 

He went on to describe a "conceptual driver" for PCIe endpoints that is based on the students' work but adds on things like DMA as well as additional virtio drivers. Adding a virtio block device would allow embedded devices to use hard disks over PCIe or, by implementing a Plan 9 filesystem (9pfs) virtio driver, individual files could be used directly over PCIe. All of this depends on using the virtio abstraction. 

Virtio is seen as a useful layer in the driver because it is a standard abstraction for ""doing something when you aren't limited by hardware"". Networking, block device, and filesystem "hosts" are all implemented atop virtio drivers, which makes them available fairly easily. One problem area, though, is the runtime configuration piece. The problem there is ""not in coming up with something that works, but something that will also work in the future"". 

Replacing the `ioctl()` interface with the InfiniBand verbs (ibverb) interface is planned. The ibverb interface may not be the best choice in an abstract sense, but it exists and supports [OpenMPI](<http://www.open-mpi.org/>), so the new driver should implement it as well. 

Two types of virtqueue implementations are envisioned, one for memory-mapped I/O (MMIO) and the other for a DMA-based virtqueue. The MMIO would be the most basic virtqueue implementation, with a local read of a remote write. Read access on PCIe is much slower than write because a read must flush all writes then wait for data reception. Data and signaling information would have separate areas so that data ordering guarantees could be relaxed on the data area for better performance, while strict data ordering would be set for the signalling area. 

The DMA engine virtqueue implementation would be highly hardware-specific to incorporate performance and other limitations of the underlying engine. In some cases, for example, it is not worth setting up a DMA for transfers of less than 2K, so copying via MMIO should be used instead. DMA would be used for transferring payload data, but signaling would still be handled via MMIO. Bergmann noted that the kernel DMA abstraction may not provide all that is needed so enhancements to that interface may be required as well. 

Bergmann did not provide any kind of time frame in which this work might make its way into the kernel as it is a work in progress. There is much still to be done, but his presentation laid out a roadmap of where he thinks it is headed. 

In a post-talk email exchange, Bergmann points to his [triblade-2.6.27](<http://git.kernel.org/?p=linux/kernel/git/arnd/cell-2.6.git;a=shortlog;h=triblade-2.6.27>) branch for those interested in looking at the current state of affairs, while noting that it ""is only mildly related to what I think we should be doing"". He also mentioned a [patch by Ira Snyder](<http://lwn.net/Articles/307774/>) that implements virtual ethernet over PCI, which ""is more likely to go into the kernel in the near future"". Bergmann and Snyder have to agreed to join forces down the road to add more functionality along the lines that were outlined in the talk.
