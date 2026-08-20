---
title: "Using dma-bufs for read and write operations"
url: https://lwn.net/Articles/1072317/
date: "May 12, 2026"
category: io uring
author: "By Jonathan Corbet May 12, 2026 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 12, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

The kernel's [dma-buf subsystem](<https://docs.kernel.org/driver-api/dma-buf.html>) provides a way for drivers to share memory buffers, usually in order to support efficient device-to-device I/O. At the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Pavel Begunkov, assisted by Kanchan Joshi, led a joint session of the storage and memory-management tracks to explore ways to make the use of dma-bufs more efficient yet, and to make them available for read and write operations initiated by user space. 

Begunkov began with a mention of [this 2022 patch set](<https://lwn.net/ml/io-uring/20220805162444.3985535-1-kbusch@fb.com/>) from Keith Busch, which pointed out that, while a dma-buf can facilitate efficient I/O operations, there is often a fair amount of expensive setup work to do before those operations happen. This work includes the creation of various internal data structures, the establishment of DMA mappings, and possibly some expensive configuration of the I/O memory-management unit (IOMMU). When a new dma-buf must be created for each operation, that work must be repeated and much of the efficiency is lost. Busch's solution was to allow dma-bufs to be registered with the [io_uring](<https://man7.org/linux/man-pages/man7/io_uring.7.html>) subsystem, similarly to how io_uring supports registered files and buffers. That would allow the registered dma-buf to be reused (within io_uring), spreading the setup cost across multiple operations. 

[![\[Pavel Begunkov\]](https://static.lwn.net/images/conf/2026/lsfmm/PavelBegunkov-sm.png)](<https://lwn.net/Articles/1072331/>) That series never made it into the mainline, but interest in that concept remains. Begunkov has [a patch series of his own](<https://lwn.net/ml/io-uring/cover.1763725387.git.asml.silence@gmail.com/>) extending Busch's work. His objective, he said in the session, is to create a consistent infrastructure to allow for the use of dma-bufs in the networking and storage subsystems. He has chosen io_uring registered buffers as the user-space API, with a special registration operation needed for dma-bufs. User space would obtain a dma-buf from a subsystem that supports them, then register the associated file descriptor with io_uring; thereafter, it would be available for I/O. 

There are some requirements for this work. Despite the use of io_uring as the API, the internals of this mechanism should not be io_uring-specific; it should eventually be extendable to filesystems and beyond. It also has to support map invalidation by the dma-buf provider. The internal API is centered around a new `io_dmabuf_token` structure, which is the interface between the driver implementing the dma-buf and io_uring. Specific I/O requests are tracked with an `io_dmabuf_map` structure, which is supported by the [iomap subsystem](<https://docs.kernel.org/filesystems/iomap/index.html>) to provide a driver-specific way of iterating through I/O requests. The patch series is coming along, but is not yet ready. 

One question that comes up occasionally, he said, is whether [P2PDMA](<https://lwn.net/Articles/767281/>) should be used for this purpose. There are a few reasons why P2PDMA is not sufficient. It is unable to use dma-bufs that user space may already have, but that is a requirement. The new API can support cheaper intermediate transformations of data, better optimize IOMMU use, and provide support for map invalidation; a member of the audience said that P2PDMA supports map invalidation as well. The downside of not using P2PDMA is, of course, the need for a new API, and one that is limited to io_uring for now. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Use cases, Begunkov said, include applications that need to optimize IOMMU use with normal host memory. There are a number of networked storage solutions that could benefit from easy movement of data between network interfaces and filesystems. There is also evidently a company that wants to use this feature for its GPU infrastructure. Joshi added that the NVMe subsystem could benefit from this feature to implement pass-through support, among other things. Future plans include adding support for more block drivers, for the SCSI subsystem, and for filesystems. 

An IOMMU pre-mapping benchmark showed performance improvements of up to 8.8x. Notably, pre-mapping completely eliminated the performance penalty that comes from using the IOMMU in either the lazy or strict modes, both of which do a certain amount of TLB invalidation on mapping changes to enforce device isolation. In other words, it is no longer necessary to use the IOMMU pass-through mode, which is seen by some as being less secure, to get full performance 

Jason Gunthorpe, though, wondered why pass-through mode was not enough, and how the additional complexity of pre-mapping was justified; Begunkov answered that security concerns were behind the desire to get away from pass-through mode. Gunthorpe said that a better solution was to just not leave the IOMMU mapped after operations are complete. Christoph Hellwig said that some sites are requiring IOMMU use, and that the memory coalescing that IOMMUs do is helpful for performance, so full IOMMU support with good performance is needed; Gunthorpe acknowledged that those were good points. Matthew Wilcox suggested that the mapping of a buffer is a good time to defragment the underlying memory, removing the need for coalescing in the first place. 

David Howells worried that misuse (accidental or deliberate) of dma-bufs could create problems by clogging all of the available IOMMU slots, and wondered whether this feature would require privilege to use. Begunkov agreed that it could be a problem, and said that some sort of capability check would be required. 

Christian Brauner took issue with the fact that this feature uses scatterlists, an internal API that the developers would eventually like to get rid of; Hellwig answered that dma-bufs still need scatterlists, so they cannot be avoided for now. There was some unfocused discussion on removing the scatterlist dependency from dma-bufs, but Hellwig said that Begunkov's work should not be held up waiting for that cleanup to be done. As time ran out, there was also some discussion of how filesystem access might be supported; patches for that have not yet been seen.
