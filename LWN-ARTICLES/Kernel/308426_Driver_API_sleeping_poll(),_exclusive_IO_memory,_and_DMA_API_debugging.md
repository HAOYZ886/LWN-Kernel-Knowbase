---
title: "Driver API: sleeping poll(), exclusive I/O memory, and DMA API debugging"
url: https://lwn.net/Articles/308426/
date: "November 24, 2008"
category: "Device drivers-Support APIs; Direct memory access; IO memory; poll"
author: "By Jonathan Corbet November 24, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 24, 2008

There are currently a number of proposed driver API changes being discussed on the lists. None of them are major, but they are worth being aware of. 

### poll()

Most of the functions in the `file_operations` structure are concerned with I/O. So it is not surprising that these functions are allowed to sleep. Except that, as it turns out, one of them - `poll()` \- cannot. There is nothing inherent in the `poll()` or `select()` system calls which would require the driver `poll()` callback to be nonblocking; this requirement is, instead, a result of the implementation. In essence, the core `poll()` implementation looks like this: 

```
for (;;)
            set_current_state(TASK_INTERRUPTIBLE)
        	for each fd to poll
    	    ask driver if I/O can happen
    	    add current process to driver wait queue
            if one or more fds are ready
    	    break
     	schedule_timeout_range(...)
```

The problem is relatively straightforward: if a specific driver chooses to sleep in its `poll()` callback, the current task state will get set back to TASK_RUNNING and `schedule_timeout_range()` will return immediately. So a sleeping driver turns the main loop into a busy-wait. 

The [solution](<http://lwn.net/Articles/308243/>), as developed by Tejun Heo, is also straightforward. His patch causes `sys_poll()` to define a custom wakeup function which, in turn, sets a new `triggered` flag when called. That eliminates the need to put the process into `TASK_INTERRUPTIBLE` for the duration of the main loop; that can be done, instead, right before actually sleeping. 

Most driver writers can remain unaware of this change, which looks highly likely to be merged for 2.6.29. But, for those who need it, there will be one more degree of flexibility in the implementation of `poll()` callbacks. 

### Exclusive I/O memory

For a while, developers involved in the [hunt for the e1000e corruption bug](<http://lwn.net/Articles/304105/>) thought that the X server might be the problem. The real bug turned out to be elsewhere, but the suspicion cast upon X led to the development of a new API designed to make it harder for user-space programs to interfere with the operation of an in-kernel driver. 

In particular, it seemed sensible to prevent user space from manipulating I/O memory which has been allocated by device drivers. This can be achieved by not allowing an `mmap()` call on `/dev/mem` to map regions already given to drivers. If the `STRICT_DEVMEM` configuration option is set, the kernel will protect its own memory from mapping by user space; protecting I/O memory is really just a matter of extending that mechanism. 

Arjan van de Ven has implemented that feature in his [MMIO exclusivity](<http://lwn.net/Articles/308348/>) patch. He chose, however, not to make this protection the default. Instead, drivers which want exclusive access to an I/O memory region should call one of these new functions: 

```
int pci_request_region_exclusive(struct pci_dev *pdev, int bar, 
                                         const char *res_name);
        int pci_request_regions_exclusive(struct pci_dev *pdev, 
                                          const char *res_name);
        int pci_request_selected_regions_exclusive(struct pci_dev *pdev,
    				               int bars, 
    					       const char *res_name);
```

There is also a new, low-level allocation macro: 

```
request_mem_region_exclusive(start, n, name);
```

In each case, these functions are equivalent to their non-exclusive cousins, except for the changed name and the resulting exclusive allocation. 

There may be cases where a developer wants to be able to map a region from user space on a development system, regardless of what the driver thinks. For such situations, there is a new `iomem=relaxed` boot parameter. When `relaxed` is selected, exclusive allocations are not enforced. Clearly this is not an option which one would want to set on a production system, but it may be useful in development environments. 

### DMA API debugging

The last topic is not actually an API change, but it's worth a look anyway. The kernel provides a nice API for setting up DMA operations. In many cases, the associated functions do little or no work; the system they are running on does not require any additional effort. The result is that a lot of "tested" driver code may, in fact, have serious errors in its use of the DMA API. When those drivers are run on a different system - one with an I/O memory management unit (IOMMU) in particular - those errors could lead to no end of unpleasant behavior. 

Kernel developers like the idea of finding bugs before they bite users on remote systems. To help make that happen with the DMA API, Joerg Roedel has posted [a new DMA API debugging facility](<http://lwn.net/Articles/308237/>). This feature, when built into the kernel, should make it possible to find a number of previously-hidden bugs in device drivers. It has, in fact, already turned up a few problems with in-tree drivers, mostly in the networking subsystem. 

Use of this facility simply requires enabling a configuration option; the API itself does not change. Once it's enabled, this code will check for a number of problems, including freeing DMA buffers with a different size than was given at allocation time, freeing buffers which were never allocated at all, mixing coherent and non-coherent functions on the same buffer, confusion over I/O directions, and more. Each of these problems might slip by on a developer's test system, but might create havoc where an IOMMU is being used. When a problem is found, a warning and stack traceback are logged. 

The response to this API has been positive. The biggest complaint seems to be about the fact that this API is implemented as an x86-specific feature. So it will probably have to be made generic before merging - after all, developers on other platforms are entirely capable of introducing DMA-related bugs too. Once it goes in, this feature should probably be enabled on any system used for driver development.
