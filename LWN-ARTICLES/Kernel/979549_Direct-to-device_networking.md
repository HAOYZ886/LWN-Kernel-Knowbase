---
title: "Direct-to-device networking"
url: https://lwn.net/Articles/979549/
date: "June 27, 2024"
category: Networking
author: "By Jonathan Corbet June 27, 2024"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 27, 2024

It has been nearly one year since [the first version](<https://lwn.net/ml/netdev/20230710223304.1174642-1-almasrymina@google.com/>) of the device memory TCP patches was posted by Mina Almasry. Now on [the 14th revision](<https://lwn.net/ml/all/20240625195407.1922912-1-almasrymina@google.com>), this series appears to be stabilizing. Device memory TCP is a specialized networking feature requiring a certain amount of setup, but it could provide a significant performance improvement for some data-intensive applications. 

The kernel's networking stack is designed to manage data transfer between the system's memory and a network device. Much of the time, data will be copied into a kernel buffer on its way to or from user space; in some cases, there are zero-copy options that can accelerate the process. But even zero-copy operations can be wasteful when the ultimate source or sink for the data is a peripheral device. An application that is, for example, reading data from a remote system and feeding it into a device like a machine-learning accelerator may never actually look at the data it is moving. 

Copying data through memory in this way can be expensive, even if the copies themselves are done with DMA operations and (mostly) do not involve the CPU. Memory bandwidth is limited; copying a high-speed data stream into and out of memory will consume much of that bandwidth, slowing down the system uselessly. If that data could be copied directly between the network device and the device that is using or generating that data, the operation would run more quickly and without the impact on the performance of the rest of the system. 

Device memory TCP is intended to enable this sort of device-to-device transfer when used with suitably capable hardware. The feature is anything but transparent — developers must know exactly what they are doing to set it up and use it — but for some applications the extra effort is likely to prove worthwhile. Although some of the changelogs in the series hint at the ability to perform direct transfers of data in either direction, only the receive side (reading data from the network into a device buffer) is implemented in the current patch set. 

The first step is to assemble a direct communication channel between the network device and the target device using the [dma-buf mechanism](<https://docs.kernel.org/driver-api/dma-buf.html>). The device that is to receive the data must have an API (usually provided via [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>)) to create the dma-buf, which will be represented by a file descriptor. A typical application is likely to create several dma-bufs, so that data reception and processing can happen in parallel, to set up a data pipeline. Almasry's patch set adds a new netlink operation to bind those dma-bufs to a network interface, thus establishing the connection between the two devices. 

Some system-configuration work is required as well. High-performance network interfaces usually have multiple receive queues; the dma-bufs can be bound to one or more of those queues. For the intended data stream to work correctly, the interface should be configured so that only the traffic of interest goes to the queue(s) that have the dma-bufs bound to them, while any other traffic goes to the remaining receive queues. 

The dma-buf binding will make a range of device memory available to the network interface. A new memory allocator has been added to manage that memory and make it available for incoming data when the user specifies that the data should be written directly to device memory. To perform such an operation, the application should call `[recvmsg()](<https://man7.org/linux/man-pages/man2/recvmsg.2.html>)` with the `MSG_SOCK_DEVMEM` flag. An attempt to read data that has been directed to the special receive queue(s) without that flag will fail with an EFAULT error. 

If the call succeeds, the data that was read will have been placed somewhere in device memory where the application may not be able to see it. To find out what has happened, the application must look at the [control messages](<https://man7.org/linux/man-pages/man3/cmsg.3.html>) returned by `recvmsg()`. A `SCM_DEVMEM_DMABUF` control message indicates that the data was delivered into a dma-buf, and provides the ID of the specific buffer that was used. If, for some reason, it was not possible to copy the data directly into device memory, the control message will be `SCM_DEVMEM_LINEAR`, and the data will have been placed in regular memory. 

After the operation completes, the application owns the indicated dma-buf; it can proceed to inform the device that some new data has landed there. Once that data has been processed, the dma-buf can be handed back to the network device with the `SO_DEVMEM_DONTNEED` [`setsockopt()`](<https://man7.org/linux/man-pages/man2/getsockopt.2.html>) call. This should normally be done as quickly as possible, lest the interface run out of buffers for incoming packets and start dropping them — an outcome that would defeat the performance goals of this entire exercise. 

[This documentation patch](<https://lwn.net/ml/all/20240613013557.1169171-13-almasrymina@google.com>) in the series gives an overview of how the device memory TCP interface works. It also documents a couple of quirks it introduces due to the fact that any packet data that is written directly to device memory is unreadable by the kernel. For example, the kernel cannot calculate or check any checksums transmitted with the data; that checking has long been offloaded to network devices anyway, so this should not be a problem. Perhaps a more significant limitation is that any sort of packet filtering that depends on looking at the payload, including filters implemented in BPF, cannot work with device memory TCP. 

The patch set includes [a sample application](<https://lwn.net/ml/all/20240613013557.1169171-14-almasrymina@google.com>), an implementation of `netcat` using dma-bufs from [udmabuf](<https://lwn.net/ml/all/20180703075359.30349-1-kraxel@redhat.com/>). The series does not include any implementation for an actual network device; Almasry maintains [a repository](<https://github.com/mina/linux/commits/tcpdevmem-v13/>) containing an implementation for the [Google gve driver](<https://www.kernel.org/doc/html/next/networking/device_drivers/ethernet/google/gve.html>). This work has evolved considerably over the last year, but it appears to be settling down and might just find its way into the mainline in the relatively near future.
