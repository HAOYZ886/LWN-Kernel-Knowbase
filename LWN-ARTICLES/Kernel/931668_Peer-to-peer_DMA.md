---
title: "Peer-to-peer DMA"
url: https://lwn.net/Articles/931668/
date: "May 16, 2023"
category: Compute Express Link CXL; NVMe
author: "By Jake Edge May 16, 2023 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 16, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

In a plenary session on the first day of the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Stephen Bates led a discussion about [peer-to-peer DMA](<https://lwn.net/Articles/767281/>) (P2PDMA). The idea is to remove the host system's participation in a transfer of data from one PCIe-connected device to another. The feature was originally aimed at NVMe SSDs so that data could simply be copied directly to and from the storage device without needing to move it to system memory and then from there to somewhere else. 

#### Background

The idea goes back to 2012 or so, Bates said, when he and Logan Gunthorpe (who did "most of the real work") were working on NVMe SSDs, RDMA, and NVMe over fabrics (before it was a standard, he thought). Some customers suggested that being able to DMA directly between devices would be useful. With devices that exposed some memory (which would be called a "controller memory buffer" or CMB today) they got the precursor to P2PDMA working. There are some user-space implementations of the feature, including for [SPDK](<https://spdk.io/>) and NVIDIA's [GPUDirect Storage](<https://developer.nvidia.com/blog/gpudirect-storage/>), which allows copies directly between NVMe namespaces and GPUs. 

[ ![\[Stephen Bates\]](https://static.lwn.net/images/2023/lsfmb-bates-sm.png) ](<https://lwn.net/Articles/931947/>)

Traditional DMA has some downsides when moving data between two PCIe devices, such as an NVMe SSD and an RDMA network card. All of the DMA operations come into the system memory from one of the devices, then have to be copied out of the system memory to the other device, which doubles the amount of memory-channel bandwidth required. If user-space applications are also trying to access the RAM on the same physical DIMM as the DMA operation, there can be various quality-of-service problems as well. 

P2PDMA avoids those problems, but comes with a number of challenges, he said. The original P2PDMA implementation for Linux was in-kernel-only; there were some hacks that allowed access from user space, but they were never merged into the mainline. More recently, though, the 6.2 kernel has support for user-space access to P2PDMA, at least in some circumstances. P2PDMA is available in the NVMe driver but only devices that have a CMB can be a DMA source or destination. NVMe devices are the only systems currently supported as DMA masters as well. 

Bates is unsure whether Arm64 is a fully supported architecture currently, as there is "some weird problem" that Gunthorpe is working through, but x86 is fully supported. An IOMMU plays a big role for P2PDMA because it needs to translate physical and virtual addresses of various sorts between the different systems; "believe me, DMAing to the wrong place is never a good thing". The IOMMU can also play a safeguard role to ensure that errant DMA operations are not actually performed. 

Currently, there is work on allowlists and blocklists for devices that do and do not work correctly, but the situation is generally improving. Perhaps because of the GPUDirect efforts, support for P2PDMA in CPUs and PCIe devices seems to be getting better. He pointed to his [p2pmem-test repository](<https://github.com/sbates130272/p2pmem-test>) for the user-space component that can be used to test the feature in a virtual machine (VM). As far as he knows, no other PCIe drivers beyond the NVMe driver implement P2PDMA, at least so far. 

#### Future

Most NVMe drivers are block devices that are accessed via logical block addresses (LBAs), but there are devices with object-storage capabilities as well. There is also a computational storage interface coming soon (which was the topic of the next session) for doing computation (e.g. compression) on data that is present on a device. NVMe namespaces for byte-addressable storage are coming as well; those are not "load-store interfaces", which would be accessible from the CPU via load and store instructions as with RAM, but are instead storage interfaces available at byte granularity. Supporting P2PDMA for the NVMe persistent memory region (PMR), which is load-store accessible and backed by some kind of persistent data (e.g. battery backed-up RAM), is a possibility on the horizon, though he has not heard of any NVMe PMR drives in development. PMR devices could perhaps overlap the use cases of [CXL](<https://www.computeexpresslink.org/>), he said. 

Better VM and IOMMU support is in the works. PCIe has various mechanisms for handling and caching memory-address translations, which could be used to improve P2PDMA. Adding more features to QEMU (e.g. [SR-IOV](<https://en.wikipedia.org/wiki/Single-root_input/output_virtualization>)) is important because it is difficult to debug problems using real hardware. Architecture support is also important; there may still be problems with Arm64 support, but there are other important architectures, like RISC-V, that need to have P2PDMA support added. 

CXL had been [prominently featured in the previous session](<https://lwn.net/Articles/931282/>), so Bates said he wanted to dig into it a bit. P2PDMA came about in a world where CXL did not exist, but now that it does, he thinks there are an interesting set of use cases for P2PDMA in a CXL world. Electrically and physically, CXL is the same as PCIe, which means that both types of devices can plug into the same bus slots. They are different at the data link layer, but work has been done on [CXL.io](<https://en.wikipedia.org/wiki/Compute_Express_Link#Protocols>), which translates PCIe to CXL. 

That means that an NVMe drive that has support for CXL [flow-control units](<https://en.wikipedia.org/wiki/Flit_\(computer_networking\)>) (flits) can be plugged into a CXL port and can then be used as a storage device via the NVMe driver on the host. He and a colleague had modeled that using QEMU the previous week, which may be the first time it had ever been done. He believes it worked but more testing is needed. 

Prior to CXL 3.0, doing P2PDMA directly between CXL memory and an NVMe SSD was not really possible because of cache-coherency issues. CXL 3.0 added a way for CXL to tell the CPUs that it was about to do DMA for a particular region of physical memory and ask the CPUs to update the CXL memory from their caches. The unordered I/O (UIO) feature added that ability, which can be used to move large chunks of data from or to storage devices at hardware speeds without affecting the CPU or its memory interconnects. Instead of a storage device, an NVMe network device could be used to move data directly out of CXL memory to the network. 

Bates said that peer-to-peer transfers of this sort are becoming more and more popular, though many people are not using P2PDMA to accomplish them. That popularity will likely translate to more users of P2PDMA over time, however. At that point, LSFMM+BPF organizer Josef Bacik pointed out that time had expired on the slot, so the memory-management folks needed to head off to [their next session](<https://lwn.net/Articles/931406/>), while the storage and filesystem developers continued the discussion. 

David Howells asked if Bates had spoken with graphics developers about P2PDMA since it seems like they might be interested in using it to move, say, textures from storage to a GPU. Bates said that he had been focusing on cloud and enterprise kinds of use cases, so he had not contacted graphics developers. The large AI clusters are using peer-to-peer transfers to GPUs, though typically via the GPUDirect mechanism. 

The NVMe community has been defining new types of namespaces lately, Bates said. The LBA namespace is currently used 99% of the time, but there are others coming as he had noted earlier. All of those namespace types and command sets can be used over both PCIe and CXL, but they can also be used over fabrics with RDMA or TCP/IP. Something that is not yet in the standard, but he hopes is coming, is providing a way to present an NVMe namespace (or a sub-region of it) as a byte-addressable, load-store region that P2PDMA can then take advantage of. 

There was a digression on what was meant by load-store versus DMA for these kinds of operations. Bates said that for accessing data on a device, DMA means that some kind of descriptor is sent to a data-mover that would simply move the data as specified, whereas load-store means that a CPU is involved in doing a series of load and store operations. So there would be a new NVMe command requesting that a region be exposed as a CMB, a PMR, or "something new that we haven't invented yet"; the CPU (or some other device, such as a DMA data-mover) can then do load-store accesses on the region. 

One use case that he described would be having an extremely hot (i.e. frequently accessed) huge file on an NVMe drive, but wanting to be able to access it directly with loads and stores. A few simple NVMe commands could prepare this data to be byte-accessible, which could then be mapped into the application's address space using [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>); it would be like having the file in memory without the possibility of page faults when accessing it.
