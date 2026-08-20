---
title: An operations structure for swap devices
url: https://lwn.net/Articles/1083094/
date: "July 23, 2026"
category: "Memory management-Swapping"
author: "By Jonathan Corbet July 23, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 23, 2026

One of the ideas raised at the [2026 Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2026/>) (LSFMM+BPF) was the creation of [an operations structure](<https://lwn.net/Articles/1072657/#:~:text=Abstracting%20the%20swap%20backend>) for the swap subsystem. Like many parts of the kernel, the swap layer evolved over time, with pieces being added as needed; the end result of this evolution is rarely what one would expect had the subsystem been designed today. The interface between the swap layer and the devices it uses is just one example. It appears that one result of the swap subsystem's evolution — the lack of an abstraction layer to interface with underlying storage — will soon be addressed, but in a different way than was initially envisioned. 

In the early days, the kernel was only able to swap to a dedicated partition on a local disk drive. If that partition turned out to be too small — and it always turned out that way in the end — the only way to fix it was, normally, to repartition the entire disk. The eventual addition of the ability to use a file within a filesystem as swap space made life easier for system administrators; later still, the kernel gained the ability to swap to networked filesystems. Within the swap layer, each of these possible swap backends is coded as a set of special cases where their behavior differs. This lack of abstraction at that level makes the swap subsystem harder to maintain and improve. 

A [a patch series](<https://lwn.net/ml/all/20260515015737.890994-1-baoquan.he@linux.dev>) meant to improve this situation, created by Baoquan He, had been in the works for some time. The interface between the swap layer and the backend would be defined by this structure: 

```
struct swap_ops {
    	void (*read_folio)(struct swap_info_struct *sis, struct folio *folio,
    			   struct swap_iocb **plug);
    	void (*write_folio)(struct swap_info_struct *sis, struct folio *folio,
    			    struct swap_iocb **plug);
    	void (*unplug)(struct swap_iocb *sio);
        };
```

The purpose of the `read_folio()` and `write_folio()` functions is fairly obvious: they are called to move one or more pages between memory and the underlying swap device. The `unplug()` function existed to start a batch of accumulated I/O operations. Together, this set of operations was able to hide the differences between swap backends from the rest of the swap subsystem. 

In He's series, there were three `swap_ops` structures defined, for three different backend cases: 

  * `bdev_async_swap_ops` is for the normal case of swapping to a block device. Operations happen asynchronously, since block devices can take some time to carry them out. It is worth noting that the case of swapping to a file within a local filesystem is also normally handled by these operations. When a swap file is added, the kernel asks the filesystem to specify where the file's blocks have been placed, then bypasses the filesystem for I/O thereafter. 
  * `bdev_sync_swap_ops` is also for a local block device, but the operations are synchronous; the functions will not return until the requested operations are complete. This is the "swap bypass" case, which was described in [this article](<https://lwn.net/Articles/1057102/>); in short, for fast, memory-based devices, the operations come down to memory copies, so there is no point in performing them asynchronously. 
  * `bdev_fs_swap_ops` is a set of operations that call into a filesystem to perform the actual I/O. As noted above, this case is not for local filesystems. It _is_ needed, though, for swapping to files on network filesystems. 

This series had been through seven revisions and appeared to be on track for merging into the mainline. After LSFMM+BPF, though, Christoph Hellwig showed up with [a somewhat different approach](<https://lwn.net/ml/all/20260713093350.2154226-1-hch@lst.de>) to the problem. His series is focused on increasing the batching of swap-I/O operations for better performance. As part of that effort, it adds an operations structure that is inspired by He's work, but which takes a different approach: 

```
struct swap_ops {
    	unsigned int flags;
    	bool (*can_merge)(struct folio *folio, struct folio *prev_folio,
    			size_t prev_folio_size, int rw);
    	void (*submit_write)(struct swap_io_ctx *ctx);
    	void (*submit_read)(struct swap_io_ctx *ctx);
        };
```

The `can_merge()` function determines whether the I/O operations for two folios can be merged into a single, larger operation. It has to determine whether the storage for the two folios is logically adjacent on the backing store, so the calculation must be different for different backing devices. `submit_write()` and `submit_read()` are for initiating I/O, with the details of the operation gathered together into the `ctx` argument. There is only one possible `flags` value, `SWAP_OPS_F_REQUIRE_NOFS`, which indicates that any reclaim operations must not call into the filesystem, since that could generate a recursive call into the filesystem hosting the swap file. 

In Hellwig's patch set, there are only two operations structures defined: `swap_bdev_ops` for block-device backends, and `swap_fs_ops` for filesystem-backed backends. The synchronous swap-bypass case is handled with tests in the block-device implementation, as with current kernels. 

This series is on its fifth revision, and seems to be mostly ready, though it may not come together in time for the 7.3 merge window. This work, it seems, has run into a problem that is showing up with increasing frequency in the kernel community: the Sashiko review tool has few complaints about the patches themselves, but [finds a couple of pre-existing problems](<https://sashiko.dev/#/patchset/20260713093350.2154226-1-hch@lst.de>) in the code that the patches change. There is a natural desire to see those problems fixed, but delaying new work to fix them is not always a popular decision. Whether that will happen in this case is yet to be determined.
