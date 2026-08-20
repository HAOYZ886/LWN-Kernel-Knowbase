---
title: Destaging ION
url: https://lwn.net/Articles/792733/
date: "July 9, 2019"
category: "Device drivers-Support APIs; Direct memory access; ION"
author: "July 9, 2019 This article was contributed by Marta Rybczyńska"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

July 9, 2019

This article was contributed by Marta Rybczyńska

The Android system has shipped a couple of allocators for DMA buffers over the years; first came [PMEM](<https://elinux.org/Android_Kernel_Features#pmem>), then its replacement [ION](<https://lwn.net/Articles/480055/>). The ION allocator has been in use since around 2012, but it remains stuck in the kernel's staging tree. The work to [add ION to the mainline](<https://lwn.net/Articles/565469/>) started in 2013; at that time, the allocator had multiple issues that made inclusion impossible. Recently, John Stultz [posted a patch set](<https://lwn.net/ml/linux-kernel/20190624194908.121273-1-john.stultz@linaro.org/>) introducing DMA-BUF heaps, an evolution of ION, that is designed to do exactly that -- get the Android DMA-buffer allocator to the mainline Linux kernel. 

Applications interacting with devices often require a memory buffer that is shared with the device driver. Ideally, it would be memory mapped and physically contiguous, allowing direct DMA access and minimal overhead when accessing the data from both sides at the same time. ION's main goal is to support that use case; it implements a unified way of defining and sharing such memory buffers, while taking into account the constraints imposed by the devices and the platform.

ION provides a number of memory pools, called "heaps", that have different properties, like whether they are physically contiguous. These heaps include the "system" heap, containing memory allocated via `vmalloc_user()`, the "system_contig" heap, which uses `kzalloc()`, and the "carveout" heap, managing a physically contiguous region set aside at boot. The user-space API allows applications to allocate, free, and share buffers from any of these heaps.

ION was developed, out of tree, in parallel with in-tree kernel APIs like [DMA buffer sharing (DMA-BUF)](<https://lwn.net/Articles/474819/>) and the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>). It naturally duplicates parts of their functionality. In addition, as ION's first platform was Android on 32-bit ARM processors, it used ARM-specific kernel APIs when there were no generic ones available. This obviously did not help the upstreaming process. The new DMA-BUF heaps patch set is a complete rework of the ION internals: it uses CMA to implement a physically contiguous heap from a special memory zone and it does not make use of any architecture-specific functions. [A self-test included](<https://lwn.net/ml/linux-kernel/20190607030719.77286-6-john.stultz@linaro.org/>) with the patch set presents the API.

#### Heaps and allocations

Each heap is represented by a special file in the `/dev/dma_heap` directory; an application will open a specific heap file to be able to allocate from that heap. The allocations are done using the `DMA_HEAP_IOC_ALLOC` `ioctl()` on the resulting file descriptor. This command takes one parameter, a pointer to a `dma_heap_allocation_data` structure: 

```
struct dma_heap_allocation_data {
            __u64 len;
            __u32 fd;
            __u32 fd_flags;
            __u64 heap_flags;
            __u32 reserved0;
            __u32 reserved1;
        };
```

`len` is the size of the desired allocation in bytes. `fd` should be set to zero when setting up the structure; it will be filled in by the `DMA_HEAP_IOC_ALLOC` operation with a file descriptor representing the allocated DMA-BUF. `fd_flags` describes how the file descriptor will be set up (the valid flags are `O_CLOEXEC`, `O_RDONLY`, `O_WRONLY`, and `O_RDWR`) and `heap_flags` stores the flags passed to the heap allocator; it should be set to zero. Finally, there are two reserved fields that should also be set to zero. The `ioctl()` returns zero when the allocation is successful.

To access the allocated memory, the application needs to call `mmap()` on the returned buffer file descriptor. When the buffer is no longer needed, the user code should close its file descriptor, which will free the allocated memory.

To summarize, each heap used by an application has an associated file descriptor, as does every allocated buffer. The buffer handles in DMA-BUF heaps are generic DMA-BUF handles that can be passed to drivers that understand such buffers. This API differs from the original ION approach, where there was one centralized special file for the allocator itself, and separate non-standard handles for the buffers. There was also a specific `ioctl()` to share the memory that does not exist in DMA-BUF heaps.

#### Memory access synchronization

One of the complex issues when handling a buffer that is shared between devices and CPUs is deciding who can access it at any given time. This is because of the caches: a processor's accesses typically involve the cache, while device accesses may not. Concurrent access may cause a mismatch between the cache and memory, leading to data corruption. To handle this issue, the drivers and applications must declare when they need to access shared memory for reading or writing; this allows the kernel to manage the caches correctly.

The original ION did not handle synchronization; DMA-BUF heaps uses the DMA-BUF API directly for this purpose. Synchronization is controlled by the `DMA_BUF_IOCTL_SYNC` `ioctl()`, which takes a structure with flags to describe the synchronization type. Before accessing the shared buffer, user code should use the `DMA_BUF_SYNC_START` flag with the required access mode (`DMA_BUF_SYNC_READ`, `DMA_BUF_SYNC_WRITE`, or `DMA_BUF_SYNC_RW`, which is a combination of the two). When the access is finished, it should use `DMA_BUF_SYNC_END` with the same access-mode flags.

#### Available heaps and adding new ones

The implementation uses a modular approach to the heaps; it defines a general framework that is used by each specific heap implementation. The patch set offers two heap types: the system heap using `alloc_page()` and the "cma" heap using the CMA allocator (if available in the system).

As in the original ION, it is up to the application developer to choose the right heap, which must correspond to the requirements of all the devices involved. This is a limitation, but the problem is complicated and currently no mainline solution exists. In embedded systems, where DMA heaps will most likely be used, the hardware configuration is fixed, so that the memory constraints of the devices are known in advance.

Kernel developers are given a framework to add more heaps, which currently can only be done at boot time. Each heap needs to fill the operations structure and the export structure. The operations structure, `struct dma_heap_ops`, is currently simple, containing a single function:

```
struct dma_heap_ops {
            int (*allocate)(struct dma_heap *heap,
                            unsigned long len,
                            unsigned long fd_flags,
                            unsigned long heap_flags);
        };
```

The export structure has the following format: 

```
struct dma_heap_export_info {
            const char *name;
            struct dma_heap_ops *ops;
            void *priv;
        };
```

`name` is the name of the heap, `ops` is the pointer to the operations structure shown above, and the `priv` is a place for heap-specific data. The parameters exactly match the structure of the `DMA_HEAP_IOC_ALLOC` `ioctl()`; the allocator should return the file descriptor of the allocated DMA-BUF.

After filling out the two structures, the heap implementation needs to add the heap using:

```
struct dma_heap *dma_heap_add(const struct dma_heap_export_info *exp_info);
```

#### Next steps

Since the time ION was first introduced, the Linux kernel has gained generic functionality that can be now used in the new implementation. The DMA-BUF heaps interface is simple, and the patch set intentionally leaves out certain functionalities (including a few heap types) and performance optimizations. Its goal is to define the API; the optimizations and new functionalities are expected to come later. A logical next step is the optimization of allocation performance. Stultz has the patches ready, but decided not to include them in the submission to simplify the review process. Allocation performance is expected to be similar to or even better than the original ION.

The patch set is currently in its sixth version and the simple, step-by-step approach seems to be working; the discussions show no major controversies. We can expect that the new API may appear in the kernel soon.
