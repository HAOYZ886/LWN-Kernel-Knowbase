---
title: "Tracing memory-mapped I/O operations"
url: https://lwn.net/Articles/270939/
date: "February 26, 2008"
category: "Development tools-mmiotrace; Reverse engineering; mmiotrace"
author: "By Jonathan Corbet February 26, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 26, 2008

Device drivers, in the end, usually do one thing: they communicate with the hardware by way of a set of memory-mapped I/O (MMIO) registers. So when one is trying to figure out what a driver is doing - for debugging purposes, perhaps - it is often interesting to look at the sequence of MMIO operations the driver performs. If one is trying to reverse-engineer a driver which is available only in binary form, watching what is done with MMIO registers may be the only way to figure out how the hardware works. To this end, the developers behind the Nouveau project developed a tool called "mmiotrace" which helps them to watch which is going on with memory-mapped I/O. Now that tool is being fixed up and pushed toward the mainline. 

Drivers gain access to MMIO regions with `ioremap()` (or one of the higher-level functions like `pci_iomap()`), so that is the logical place to hook in a tracing infrastructure. So [the current mmiotrace patch](<http://lwn.net/Articles/270647/>) adds some new variants of `ioremap()`: 

```
void __iomem *ioremap_cache_trace(unsigned long offset, unsigned long size);
        void __iomem *ioremap_nocache_trace(unsigned long offset, unsigned long size);
        void iounmap_trace(volatile void __iomem *addr);
```

These functions perform like `ioremap()` and `ioremap_nocache()`, in that they return a I/O memory pointer which can be used by the driver to get at MMIO space. What goes on internally, though, is quite different. 

On the x86 architecture (as with most others), I/O memory space is accessed with memory operations through the page tables in the usual way, so `ioremap()` just returns an address which maps onto the desired physical space. The tracing versions, though, take the extra step of marking the pages within the I/O region as not being present in the system; as a result, whenever code attempts to access that space, a page fault will be generated. 

Normally, page faults incurred when running in kernel mode will cause a kernel oops. There are exceptions, though; the functions which copy data between user and kernel space are one example. The mmiotrace patch adds another exception which tests faulting addresses against the MMIO region(s) being traced. Should the address indicate that an MMIO access is being attempted, the mmiotrace code will: 

  1. Mark the relevant page as being present in memory. 

  2. Set the TF (trace) bit in the faulting thread's processor state mask. 

  3. Invoke a "pre" handler provided by higher-level tracing code. 

  4. Indicate that the fault has been handled and return to the faulting code. 

Once all this has happened, the instruction which originally caused the page fault will be rerun, successfully this time. But the setting of the trace bit will cause a new processor trap after that instruction has been executed. At that point, the page is marked unavailable once again, the trace bit is reset (assuming it wasn't set elsewhere), the tracing layer's "post" handler is called, and life continues as normal until the next fault happens. 

The tracing layer really only has one task: figure out what the code was trying to do in MMIO space and log the action by way of the relay interface. Figuring things out means learning enough about the instruction which caused the page fault to determine which address was being accessed, whether a read or write was being performed, the size of the data being transferred, and the actual value read or written. So there is a certain amount of architecture-specific instruction grubbing code involved, which, for the current patch, is only provided for x86 machines. 

Since tracing is enabled by calling a special version of `ioremap()`, it is not possible to trace a driver module without making changes to its source and rebuilding it. That might seem like a strange requirement for a tool meant to help with reverse engineering (among other things). The driver being studied by the Nouveau project uses a GPL-licensed shim to link into the kernel, so making modifications in that case was not a hard thing to do. A more general solution may eventually need to be found, though, for situations where that sort of glue layer is not present. 

Beyond that, this patch is likely to go through a number of changes before it finds its way into the mainline. Reviewers have found a number of things which need fixing, and there's a few too many places in the code where the comments say (literally) "if this happens, all hell breaks loose." It also seems likely that mmiotrace will be merged with the recently-posted [ftrace](<https://lwn.net/Articles/270971/>) tracing mechanism. There is time to get this work done before the 2.6.26 merge window opens, but the mmiotrace hackers will need to keep the work moving forward.
