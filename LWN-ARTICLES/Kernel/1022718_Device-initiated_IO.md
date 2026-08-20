---
title: "Device-initiated I/O"
url: https://lwn.net/Articles/1022718/
date: "June 4, 2025"
category: NVMe
author: "By Jake Edge June 4, 2025 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 4, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

[Peer-to-peer DMA](<https://lwn.net/Articles/767281/>) (P2PDMA) has been part of the kernel since the [4.20 release](<https://lwn.net/Articles/775487/>) in 2018; it provides a framework that allows devices to transfer data between themselves directly, without using system RAM for the transfer. At the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), Stephen Bates led a combined storage, filesystems, and memory-management session on device-initiated I/O, which is perhaps what P2PDMA is evolving toward. Two years ago, he led a [session on P2PDMA](<https://lwn.net/Articles/931668/>) at the summit; this year's session was a brief update on P2PDMA with a look at where it may be heading. 

He began by looking at where P2PDMA is today. It started as an in-kernel API that enabled DMA requests between PCIe devices; one of the first users of the API was the NVMe-over-fabrics target, which allowed data to flow directly into an NVMe drive via remote DMA (RDMA). Access to the feature from user space was added, so that [`mmap()`](<https://www.man7.org/linux/man-pages/man2/mmap.2.html>) could be used to map device memory. That capability is being used by some companies, sometimes in conjunction with out-of-tree patches expanding the functionality. 

Bates admitted to having stepped away from P2PDMA for a bit due to a job change and other distractions, so he was not entirely sure about the status of the feature in some respects. He wondered about Arm 64-bit support; Christoph Hellwig said that it is supported, which means that architecture support is in pretty good shape, Bates said. 

Trying to do P2PDMA in the presence of an I/O memory-management unit (IOMMU) had been difficult, he said, so initially users turned off IOMMUs; that is not required any longer. There are a handful of PCIe features that have gained support, including [page request interface](<https://www.kernel.org/doc/html/next/x86/sva.html>) (PRI) and [address translation services](<https://www.intel.com/content/www/us/en/docs/programmable/683686/20-4/address-translation-services-ats.html>) (ATS); those features are meant to improve things but also end up making everything more confusing, he said. 

#### Device initiation

His goal for the session was to talk about ""what the next natural step for peer-to-peer is, and that is device-initiated I/O"". The first two days of the summit have been interesting, Bates said; he has been talking with other attendees about the speed of progress on PCIe devices, as well as the increases in I/O operations per second (IOPS) on newer NVMe SSDs. The block layer is also able to do more IOPS, with people reporting that a "hacked" io_uring can do up to 60-million IOPS per core, though Bates noted that the exact number should be taken with a grain of salt, but IOPS are increasing overall. 

[ ![\[Stephen Bates\]](https://static.lwn.net/images/2025/lsfmb-bates-sm.png) ](<https://lwn.net/Articles/1023673/>)

People are reporting that the NVMe driver can support eight-million IOPS per core, he said, in IOMMU pass-through mode, but Jens Axboe said that his testing shows around 10-12 million IOPS on a particular [Threadripper](<https://en.wikipedia.org/wiki/Threadripper>)-based system. The numbers vary widely and depend on other factors such as the temperature of the system (thus whether the CPUs are thermally limited), Bates said. The NVMe driver can only sustain around two-million IOPS when establishing a DMA mapping and programming windows into the IOMMU is required. He noted that there were some sessions at LSFMM+BPF about [improving the DMA-mapping code](<https://lwn.net/Articles/1020437/>), which may help reduce some of that overhead. 

But handling eight-million IOPS is consuming an entire CPU core to do the I/O and there are SSDs coming that can do up to ten-million IOPS. It seems a shame that people are buying fast, expensive processors that are spending all of their cycles doing I/O. There are two ways to improve that situation, he said; either reduce the number of CPU instructions needed per I/O operation or have the devices themselves issue the I/O. There is an Intel CPU instruction, which Matthew Wilcox called the ""NVMe-queue-submission instruction"", that might help, though Hellwig pointed out that "number of instructions" is not necessarily the same as time spent since instructions take a variable number of cycles. 

Since you can already do things like DMA from PCIe devices, Bates said, ""an accelerator or some other kind of I/O device that has enough intelligence to have code on it that generates NVMe-submission-queue entries and rings doorbells"" could handle its own I/O—or I/O for other devices. The smart device would run enough of the NVMe driver to do I/O requests. He noted a [paper](<https://arxiv.org/pdf/2203.04910>) that reported on what NVIDIA and the University of Illinois had done using GPUs for NVMe I/O, though it was only a proof of concept. Hellwig pointed out that Mellanox (which is now part of NVIDIA) had been doing similar things for RDMA well before that paper was written. Bates said there had been patches for that at one point, but he did not think they were merged; Jason Gunthorpe said that the feature was part of a shipping product at this point. 

Bates would like to see an open, vendor-neutral framework for device-initiated I/O ""where anyone who wants to can play in this space"". He thinks there would need to be a way to request and reserve NVMe hardware queues as part of that, though Hellwig does not think that is workable. Bates said that he does not want to take control of the device completely away from the kernel (as with [SR-IOV](<https://www.kernel.org/doc/html/latest/PCI/pci-iov-howto.html>) or [VFIO](<https://www.kernel.org/doc/html/latest/driver-api/vfio.html>)), for error-handling reasons and because there may be a need to tie it into filesystems. The administrative side would also remain in the kernel, he thought. 

If the feature was added, obviously some kind of protection is needed so that devices cannot simply read and write wherever they want. There was talk of "protection domains" for NVMe at one point, but he did not think they were added to the specification. It would be useful for NVMe to have the ability to restrict the kinds of operations that specific queues are allowed to do. For example, they might be restricted to using a particular namespace or a logical-block-address (LBA) range for a namespace. That way, the controller would ""act like a guard if the device got a little crazy"". 

Hellwig said that doing any of that requires a separate NVMe controller that can be used to shut down misbehaving devices. That is an expensive option, which is why people are looking for other solutions. Bates said that he is ""not 100% convinced"" that is the only way forward, however. 

Another thing that a ""naughty device"" might do would be to provide an invalid destination address for a DMA operation, either maliciously or mistakenly. Gunthorpe said that IOMMUs already handle that kind of problem and that code to use them is available in the kernel. 

Hellwig went into some detail of what was done for [Parallel NFS](<https://wiki.linux-nfs.org/wiki/index.php/Configuring_pNFS/spnfsd>) (pNFS) that would be applicable for this use case. It requires the use of [layout leases](<https://docs.kernel.org/admin-guide/nfs/pnfs-block-server.html>), which reserve the block layout of the file on the storage medium, but the lease is revocable by the kernel any time the layout changes. He said that he and some colleagues are considering writing a paper on using that technique for GPUs; ""you should give us some good hardware and you'll get a mention"", he said with a laugh. 

Bates said that sounded promising as a way forward for device-initiated I/O and that he had another topic that was slightly different, which he wanted to discuss in the last few minutes of his slot. There is a large accelerator company that is claiming that ""AI workloads need a lot of IOPS"", around 200-million IOPS per GPU, which is huge. The I/O operations are small, less than 512 bytes—less than a block in size—and the workload is read-only. ""Trying to do that via standard NVMe commands seems very foolish."" 

There is an NVMe base address register (BAR) for a controller memory buffer (CMB) that might be used to allow the AI accelerator to do load and store operations as with persistent memory. [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>) has something similar, Bates said. The NVMe buffer could be mapped to a particular namespace or range in a namespace and allow the accelerator to do CXL-like memory accesses to the data. Hellwig said that using the CMB was not the right approach because it would be slow, with too much command overhead; instead the BAR should be used to create a direct mapping from the namespace that is read-only. He used an NVMe device ten years ago that could do that, so a proof of concept is probably easy to put together, but getting it added to NVMe will take longer. 

At that point, the conversation split up into several as time expired.
