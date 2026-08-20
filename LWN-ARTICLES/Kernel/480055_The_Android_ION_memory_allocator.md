---
title: The Android ION memory allocator
url: https://lwn.net/Articles/480055/
date: "February 8, 2012"
category: "Android; Device drivers-Support APIs; ION; Memory management-Large allocations"
author: "February 8, 2012 This article was contributed by Thomas M. Zeng"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

February 8, 2012

This article was contributed by Thomas M. Zeng

Back in December 2011, LWN [reviewed the list of Android kernel patches](<http://lwn.net/Articles/472984/>) in the linux-next staging directory. The merging of these drivers, one of which is a memory allocator called PMEM, holds the promise that the mainline kernel release can one day boot an Android user space. Since then, it has become clear [ that PMEM is considered obsolete](<http://lists.linuxfoundation.org/pipermail/ce-android-mainline/2012-January/000031.html>) and [ will be replaced by the ION memory manager](<http://lists.linuxfoundation.org/pipermail/ce-android-mainline/2012-January/000024.html>). ION is a generalized memory manager that Google introduced in the Android 4.0 ICS (Ice Cream Sandwich) release to address the issue of fragmented memory management interfaces across different Android devices. There are at least three, probably more, PMEM-like interfaces. On Android devices using NVIDIA Tegra, there is ["NVMAP"](<http://www.spinics.net/lists/linux-tegra/msg00225.html>); on Android devices using TI OMAP, there is ["CMEM"](<http://processors.wiki.ti.com/index.php/CMEM_Overview>); and on Android devices using Qualcomm MSM, there is ["PMEM" ](<https://www.codeaurora.org/gitweb/quic/le/?p=kernel/msm.git;a=commitdiff;h=6aaa69d90a53f169afc5222684f0102356dc39c5>). All three SoC vendors are in the process of switching to ION. 

This article takes a look at ION, summarizing its interfaces to user space and to kernel-space drivers. Besides being a memory pool manager, ION also enables its clients to share buffers, hence it treads the same ground as [the DMA buffer sharing framework from Linaro](<http://lwn.net/Articles/474819/>) (DMABUF). This article will end with a comparison of the two buffer sharing schemes. 

#### ION heaps 

Like its PMEM-like predecessors, ION manages one or more memory pools, some of which are set aside at boot time to combat fragmentation or to serve special hardware needs. GPUs, display controllers, and cameras are some of the hardware blocks that may have special memory requirements. ION presents its memory pools as ION heaps. Each type of Android device can be provisioned with a different set of ION heaps according to the memory requirements of the device. The provider of an ION heap must implement the following set of callbacks: 

```
struct ion_heap_ops {
    	int (*allocate) (struct ion_heap *heap,
    			 struct ion_buffer *buffer, unsigned long len,
    			 unsigned long align, unsigned long flags);
    	void (*free) (struct ion_buffer *buffer);
    	int (*phys) (struct ion_heap *heap, struct ion_buffer *buffer,
    		     ion_phys_addr_t *addr, size_t *len);
    	struct scatterlist *(*map_dma) (struct ion_heap *heap,
    			 struct ion_buffer *buffer);
    	void (*unmap_dma) (struct ion_heap *heap, 
    	         struct ion_buffer *buffer);
    	void * (*map_kernel) (struct ion_heap *heap, 
    	         struct ion_buffer *buffer);
    	void (*unmap_kernel) (struct ion_heap *heap, 
    	         struct ion_buffer *buffer);
    	int (*map_user) (struct ion_heap *heap, struct ion_buffer *buffer,
    			 struct vm_area_struct *vma);
       };
```

Briefly, `allocate()` and `free()` obtain or release an `ion_buffer` object from the heap. A call to `phys()` will return the physical address and length of the buffer, but only for physically-contiguous buffers. If the heap does not provide physically contiguous buffers, it does not have to provide this callback. Here `ion_phys_addr_t` is a typedef of `unsigned long`, and will, someday, be replaced by `phys_addr_t` in `include/linux/types.h`. The `map_dma()` and `unmap_dma()` callbacks cause the buffer to be prepared (or unprepared) for DMA. The `map_kernel()` and `unmap_kernel()` callbacks map (or unmap) the physical memory into the kernel virtual address space. A call to `map_user()` will map the memory to user space. There is no `unmap_user()` because the mapping is represented as a file descriptor in user space. The closing of that file descriptor will cause the memory to be unmapped from the calling process. 

The default ION driver (which can be cloned from [here](<https://android.googlesource.com/kernel/common.git>)) offers three heaps as listed below: 

```
ION_HEAP_TYPE_SYSTEM:        memory allocated via vmalloc_user().
       ION_HEAP_TYPE_SYSTEM_CONTIG: memory allocated via kzalloc.
       ION_HEAP_TYPE_CARVEOUT:	carveout memory is physically contiguous and set aside at boot.
```

Developers may choose to add more ION heaps. For example, [this NVIDIA patch](<http://lists.linaro.org/pipermail/linaro-mm-sig/2012-January/001002.html>) was submitted to add ION_HEAP_TYPE_IOMMU for hardware blocks equipped with an IOMMU. 

####  Using ION from user space 

Typically, user space device access libraries will use ION to allocate large contiguous media buffers. For example, the still camera library may allocate a capture buffer to be used by the camera device. Once the buffer is fully populated with video data, the library can pass the buffer to the kernel to be processed by a JPEG encoder hardware block. 

A user space C/C++ program must have been granted access to the `/dev/ion` device before it can allocate memory from ION. A call to `open("/dev/ion", O_RDONLY)` returns a file descriptor as a handle representing an ION client. Yes, one can allocate writable memory with an `O_RDONLY` open. There can be no more than one client per user process. To allocate a buffer, the client needs to fill in all the fields except the `handle` field in this data structure: 

```
struct ion_allocation_data {
            size_t len;
            size_t align;
            unsigned int flags;
            struct ion_handle *handle;
       }
```

The `handle` field is the output parameter, while the first three fields specify the alignment, length and flags as input parameters. The `flags` field is a bit mask indicating one or more ION heaps to allocate from, with the fallback ordered according to which ION heap was first added via calls to `ion_device_add_heap()` during boot. In the default implementation, `ION_HEAP_TYPE_CARVEOUT` is added before `ION_HEAP_TYPE_CONTIG`. The flags of `ION_HEAP_TYPE_CONTIG | ION_HEAP_TYPE_CARVEOUT` indicate the intention to allocate from `ION_HEAP_TYPE_CARVEOUT` with fallback to `ION_HEAP_TYPE_CONTIG`. 

User-space clients interact with ION using the `ioctl()` system call interface. To allocate a buffer, the client makes this call: 

```
int ioctl(int client_fd, ION_IOC_ALLOC, struct ion_allocation_data *allocation_data)
```

This call returns a buffer represented by `ion_handle` which is not a CPU-accessible buffer pointer. The handle can only be used to obtain a file descriptor for buffer sharing as follows: 

```
int ioctl(int client_fd, ION_IOC_SHARE, struct ion_fd_data *fd_data);
```

Here `client_fd` is the file descriptor corresponding to `/dev/ion`, and `fd_data` is a data structure with an input `handle` field and an output `fd` field, as defined below: 

```
struct ion_fd_data {
            struct ion_handle *handle;
            int fd;
       }
```

The `fd` field is the file descriptor that can be passed around for sharing. On Android devices the `BINDER` IPC mechanism may be used to send `fd` to another process for sharing. To obtain the shared buffer, the second user process must obtain a client handle first via the `open("/dev/ion", O_RDONLY)` system call. ION tracks its user space clients by the PID of the process (specifically, the PID of the thread that is the "group leader" in the process). Repeating the `open("/dev/ion", O_RDONLY)` call in the same process will get back another file descriptor corresponding to the same client structure in the kernel. 

To free the buffer, the second client needs to undo the effect of `mmap()` with a call to `munmap()`, and the first client needs to close the file descriptor it obtained via `ION_IOC_SHARE`, and call `ION_IOC_FREE` as follows: 

```
int ioctl(int client_fd, ION_IOC_FREE, struct ion_handle_data *handle_data);
```

Here `ion_handle_data` holds the handle as shown below: 

```
struct ion_handle_data {
    	     struct ion_handle *handle;
         }
```

The `ION_IOC_FREE` command causes the handle's reference counter to be decremented by one. When this reference counter reaches zero, the `ion_handle` object gets destroyed and the affected ION bookkeeping data structure is updated. 

User processes can also share ION buffers with a kernel driver, as explained in the next section. 

#### Sharing ION buffers in the kernel

In the kernel, ION supports multiple clients, one for each driver that uses the ION functionality. A kernel driver calls the following function to obtain an ION client handle: 

```
struct ion_client *ion_client_create(struct ion_device *dev, 
                       unsigned int heap_mask, const char *debug_name)
```

The first argument, `dev`, is the global ION device associated with `/dev/ion`; why a global device is needed, and why it must be passed as a parameter, is not entirely clear. The second argument, `heap_mask`, selects one or more ION heaps in the same way as the `ion_allocation_data. The flags` field was covered in the previous section. For smart phone use cases involving multimedia middleware, the user process typically allocates the buffer from ION, obtains a file descriptor using the `ION_IOC_SHARE` command, then passes the file desciptor to a kernel driver. The kernel driver calls `ion_import_fd()` which converts the file descriptor to an `ion_handle` object, as shown below: 

```
struct ion_handle *ion_import_fd(struct ion_client *client, int fd_from_user);
```

The `ion_handle` object is the driver's client-local reference to the shared buffer. The `ion_import_fd()` call looks up the physical address of the buffer to see whether the client has obtained a handle to the same buffer before, and if it has, this call simply increments the reference counter of the existing handle. 

Some hardware blocks can only operate on physically-contiguous buffers with physical addresses, so affected drivers need to convert `ion_handle` to a physical buffer via this call: 

```
int ion_phys(struct ion_client *client, struct ion_handle *handle,
    	       ion_phys_addr_t *addr, size_t *len)
```

Needless to say, if the buffer is not physically contiguous, this call will fail. 

When handling calls from a client, ION always validates the input file descriptor, client and handle arguments. For example, when importing a file descriptor, ION ensures the file descriptor was indeed created by an `ION_IOC_SHARE` command. When `ion_phys()` is called, ION validates whether the buffer handle belongs to the list of handles the client is allowed to access, and returns error if the handle is not on the list. This validation mechanism reduces the likelihood of unwanted accesses and inadvertent resource leaks. 

ION provides debug visibility through `debugfs`. It organizes debug information under `/sys/kernel/debug/ion`, with bookkeeping information in stored files associated with heaps and clients identified by symbolic names or PIDs. 

#### Comparing ION and DMABUF

ION and DMABUF share some common concepts. The [`dma_buf` concept](<http://lwn.net/Articles/474819/>) is similar to `ion_buffer`, while `dma_buf_attachment` serves a similar purpose as `ion_handle`. Both ION and DMABUF use anonymous file descriptors as the objects that can be passed around to provide reference-counted access to shared buffers. On the other hand, ION focuses on allocating and freeing memory from provisioned memory pools in a manner that can be shared and tracked, while DMABUF focuses more on buffer importing, exporting and synchronization in a manner that is consistent with buffer sharing solutions on non-ARM architectures. 

The following table presents a feature comparison between ION and DMABUF: 

Feature |  ION  |  DMABUF   
---|---|---  
Memory Manager Role  |  ION replaces PMEM as the manager of provisioned memory pools. The list of ION heaps can be extended per device.  |  DMABUF is a buffer sharing framework, designed to integrate with the memory allocators in DMA mapping frameworks, like the work-in-progress DMA-contiguous allocator, also known as the [ Contiguous Memory Allocator](<http://lwn.net/Articles/450286/>) (CMA). DMABUF exporters have the option to implement custom allocators.   
User Space Access Control  |  ION offers the `/dev/ion` interface for user-space programs to allocate and share buffers. Any user program with ION access can cripple the system by depleting the ION heaps. Android checks user and group IDs to block unauthorized access to ION heaps.  |  DMABUF offers only kernel APIs. Access control is a function of the permissions on the devices using the DMABUF feature.   
Global Client and Buffer Database  |  ION contains a device driver associated with `/dev/ion`. The device structure contains a database that tracks the allocated ION buffers, handles and file descriptors, all grouped by user clients and kernel clients. ION validates all client calls according to the rules of the database. For example, there is a rule that a client cannot have two handles to the same buffer.  |  The [ DMA debug facility ](<http://lxr.linux.no/linux+v3.2.2/lib/dma-debug.c>) implements a global hashtable, `dma_entry_hash`, to track DMA buffers, but only when the kernel was built with the `CONFIG_DMA_API_DEBUG` option.   
Cross-architecture Usage |  ION usage today is limited to architectures that run the Android kernel.  |  DMABUF usage is cross-architecture. The [DMA mapping redesign preparation patchset](<http://lists.linaro.org/pipermail/linaro-mm-sig/2011-December/000902.html>) modified the DMA mapping code in 9 architectures besides the ARM architecture.   
Buffer Synchronization  |  ION considers buffer synchronization to be an orthogonal problem.  |  DMABUF provides a pair of APIs for synchronization. The buffer-user calls `dma_buf_map_attachment()` whenever it wants to use the buffer for DMA . Once the DMA for the current buffer-user is over, it signals 'end-of-DMA' to the exporter via a call to `dma_buf_unmap_attachment() `.   
Delayed Buffer Allocation  |  ION allocates the physical memory before the buffer is shared.  |  DMABUF can defer the allocation until the first call to `dma_buf_map_attachment()`. The exporter of DMA buffer has the opportunity to scan all client attachments, collate their buffer constraints, then choose the appropriate backing storage.   
  
ION and DMABUF can be separately integrated into multimedia applications written using [the Video4Linux2 API](<http://lwn.net/Articles/203924/>). In the case of ION, these multimedia programs tend to use PMEM now on Android devices, so switching to ION from PMEM should have a relatively small impact. 

Integrating DMABUF into Video4Linux2 is another story. It has taken [ten patches](<http://lists.linaro.org/pipermail/linaro-mm-sig/2012-January/001040.html>) to integrate the `videobuf2` mechanism with DMABUF; in fairness, many of these revisions were the result of changes to DMABUF as that interface stabilized. The effort should pay dividends in the long run because the DMABUF-based sharing mechanism is designed with DMA mapping hooks for CMA and IOMMU. CMA and IOMMU hold the promise to reduce the amount of carveout memory that it takes to build an Android smart phone. In [this email](<http://lists.linaro.org/pipermail/linaro-mm-sig/2012-January/001132.html>), Andrew Morton was urging the completion of the patch review process so that CMA can get through the 3.4 merge window. 

Even though ION and DMABUF serve similar purposes, the two are not mutually exclusive. The Linaro Unified Memory Management team has started [ to integrate CMA into ION.](<https://blueprints.launchpad.net/linaro-mm-sig/+spec/linaro-mmwg-cma-ion>) To reach the state where a release of the mainline kernel can boot the Android user space, the `/dev/ion` interface to user space must obviously be preserved. In the kernel though, ION drivers may be able to use some of the DMABUF APIs to hook into CMA and IOMMU to take advantage of the capabilities offered by those subsystems. Conversely, DMABUF might be able to leverage ION to present a unified interface to user space, especially to the Android user space. DMABUF may also benefit from adopting some of the ION heap debugging features in order to become more developer friendly. Thus far, many signs indicate that Linaro, Google, and the kernel community are working together to bring the combined strength of ION and DMABUF to the mainline kernel.
