---
title: The phaseout of the mmap() file operation
url: https://lwn.net/Articles/1038715/
date: "September 25, 2025"
category: "System calls-mmap; struct file operations"
author: "By Jonathan Corbet September 25, 2025"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 25, 2025

The [`file_operations`](<https://elixir.bootlin.com/linux/v6.16.7/source/include/linux/fs.h#L2151>) structure in the kernel is a set of function pointers implementing, as the name would suggest, operations on files. A subsystem that manages objects which can be represented by a file descriptor will provide a `file_operations` structure providing implementations of the various operations that a user of the file descriptor may want to carry out. The `mmap()` method, in particular, is invoked when user space calls the [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) system call to map the object behind a file descriptor into its address space. That method, though, is currently on its way out in a multi-release process that started in 6.17. 

The `file_operations` structure was introduced in the 0.95 release in March 1992; at that point it supported the basic `read()` and `write()` operations and not much else. Support for `mmap()` first appeared in 0.98.2 later that year, though it took a while before it actually worked as expected. The interface has evolved a bit over time, of course; in current kernels, its prototype is: 

```
int (*mmap) (struct file *, struct vm_area_struct *);
```

The [`vm_area_struct`](<https://elixir.bootlin.com/linux/v6.16.7/source/include/linux/mm_types.h#L799>) structure (usually referred to as a VMA) describes a range of a process's address space; in this case, it provides `mmap()` with information about the offset within the `file` that is to be mapped, how much is to be mapped, the intended page protections, and the address range where the mapping will be. The driver implementing `mmap()` is expected to do whatever setup is necessary to make the right thing happen when user space accesses memory within that range. There are hundreds of `mmap()` implementations within the kernel, some of which are quite complex. 

As described in [this 6.17 commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c84bf6dd2b83>) by Lorenzo Stoakes, though, there are some significant problems with this API. The `mmap()` method is invoked after the memory-management layer has done much of its setup for the new mapping. If the operation fails at the driver layer, all of that setup must be unwound, which can be a complicated task. The real problem, though, is that `mmap()` gives the driver direct access to the VMA, which is one of the core memory-management data structures. The driver can make changes to the VMA, and many do with gusto. Those changes can force the memory-management layer to redo some of its setup; worse, they can introduce bugs or create other types of unpleasant surprises. 

Over the years, a number of important memory-management structures have been globally exposed in this way; more recently, developers have been working to make more of those structures private to the memory-management code. One step in that direction is to retire the `mmap()` method in favor of a new API that more clearly constrains what code outside of the memory-management layer can do. 

#### Replacing `mmap()`

This work began with the introduction of the new `mmap_prepare()` callback in 6.17: 

```
int (*mmap_prepare)(struct vm_area_desc *);
```

That method receives a pointer to the new `vm_area_desc` structure: 

```
struct vm_area_desc {
    	/* Immutable state. */
    	struct mm_struct *mm;
    	unsigned long start;
    	unsigned long end;
    
    	/* Mutable fields. Populated with initial state. */
    	pgoff_t pgoff;
    	struct file *file;
    	vm_flags_t vm_flags;
    	pgprot_t page_prot;
    
    	/* Write-only fields. */
    	const struct vm_operations_struct *vm_ops;
    	void *private_data;
        };
```

This new method is intended to eventually replace `mmap()`; a driver cannot provide both `mmap_prepare()` and `mmap()` in the same `file_operations` structure. `mmap_prepare()` is called much earlier in the mapping process, before the VMA itself is set up. If it returns a failure status, there is a lot less work to clean up within the memory-management code. The `vm_area_desc` structure is intended to provide the driver with only the information it needs to set up the mapping, and to allow it to specify specific VMA changes to be made once the VMA itself is set up. 

Thus, for example, the driver can modify `pgoff` (the offset within the file where the mapping starts) if needed to meet alignment or other constraints. Various flags and the page protections can be changed, and the driver can provide a [`vm_operations_struct`](<https://elixir.bootlin.com/linux/v6.16.7/source/include/linux/mm.h#L579>) pointer with callbacks to handle page faults, protection changes, and other operations on the mapping. If the mapping succeeds, the memory-management layer will copy information from this structure into the VMA while keeping a grip on the overall contents of that VMA. 

#### The next step

That was the state of the API as merged for the 6.17 release; it was enough to support the conversion of a number of drivers over from `mmap()` and begin the long process of deprecating that interface. As noted above, though, some drivers do complex things in their `mmap()` implementations, and this API is not sufficient for their needs. Thus, Stoakes has been working on [an expansion of `mmap_prepare()`](<https://lwn.net/ml/all/cover.1758135681.git.lorenzo.stoakes@oracle.com>) for a wider range of use cases. 

The new capabilities are based around yet another new structure, which is added to `struct vm_area_desc` (as a field named `action`): 

```
struct mmap_action {
    	union {
    	    /* Remap range. */
    	    struct {
    		unsigned long start;
    		unsigned long start_pfn;
    		unsigned long size;
    		pgprot_t pgprot;
    	    } remap;
    	};
    	enum mmap_action_type type;
    	int (*success_hook)(const struct vm_area_struct *vma);
    	int (*error_hook)(int err);
        };
```

This structure tells the memory-management core what the driver would like to see happen _after_ the VMA has been set up and is valid. The actions defined in this patch set are `MMAP_NOTHING` (do nothing), `MMAP_REMAP_PFN`, which causes the address space covered by the VMA to be mapped to a range of page-frame numbers beginning at `start_pfn`, and `MMAP_IO_REMAP_PFN`, which performs a similar remapping into device-hosted memory. The driver could perform this remapping itself, one page at a time, in its `fault()` `vm_operations_struct` method, but it is much more efficient to just do the whole range at once. 

There are also two callbacks in that structure. The `success_hook()` callback will be called upon the successful completion of the requested action. That callback is passed a pointer to the VMA, but it is a pointer to a `const` structure, so the callback should not be able to make any changes there. This callback is [used in the `/dev/zero` driver](<https://lwn.net/ml/all/14cdf181c4145a298a2249946b753276bdc11167.1758135681.git.lorenzo.stoakes@oracle.com>) to perform a ""very unique and rather concerning"" (according to Stoakes) change that driver makes to the mapping. The `error_hook()` is called if things go wrong; it can provide a different error code to be returned as a way of filtering errors that should not make it back to user space. 

This series is in its fourth revision as of this writing; it still seems to be going through a relatively high rate of change in response to review comments. Whether it will settle in time for the 6.18 merge window is unclear at this point, so the work to remove the `mmap()` callback may have to wait another cycle before proceeding. Even after that, though, there will still be those hundreds of `mmap()` implementations to convert, so this task will not be complete for some time yet.
