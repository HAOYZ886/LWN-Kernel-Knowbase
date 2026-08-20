---
title: Connecting Linux to hypervisors
url: https://lwn.net/Articles/194543/
date: "August 8, 2006"
category: Virtualization; Xen; paravirt ops
author: 
---

Paravirtualization is the act of running a guest operating system, under control of a host system, where the guest has been ported to a virtual architecture which is _almost_ like the hardware it is actually running on. This technique allows full guest systems to be run in a relatively efficient manner. The highest-profile free paravirtualization implementation remains Xen; on the proprietary side, VMWare has been active for a long time. Both of these efforts would like to see (at least some of) their code in the mainline kernel. The kernel developers, however, are uninterested in merging a large collection of hooks specific to any one solution. 

One attempt to solve this problem, proposed by VMWare, is the [VMI interface](<http://lwn.net/Articles/175706/>). VMI works by isolating any operations which may require hypervisor intervention into a special set of function calls. The implementation of those functions is not built into the kernel; instead, the kernel, at boot time, loads a "hypervisor ROM" which provides the needed functions. The binary interface between the kernel and this loadable segment is set in stone, meaning that kernels built for today's implementations should work equally well on tomorrow's replacement. This design also allows the same binary kernel image to run under a variety of hypervisors, or, with the right ROM, in native mode on the bare hardware. 

The fixed ABI and ability to load "binary blobs" into the kernel does not sit well with all kernel developers, however. It looks like another way to put proprietary code into the kernel, which is something most kernel hackers would rather support less of. Plus, as Rusty Russell [put it](<https://lwn.net/Articles/194551/>): 

We're not good at maintaining ABIs. We're going to be especially bad at maintaining an ABI when the 99% of us running native will never notice the breakage. 

For this and other reasons, VMI has not had a smooth path into the kernel so far. That has not stopped VMWare hacker Zachary Amsden from [pushing for a binary blob interface](<http://lwn.net/Articles/194016/>) recently on linux-kernel, however. 

There have been rumblings for a while concerning an alternative hypervisor interface (called "paravirt_ops") under development. [An early implementation of paravirt_ops](<http://lwn.net/Articles/194339/>) was posted on August 7, making the shape of this interface clearer. In the end, paravirt_ops is yet another structure filled with function pointers, like many other operations structures used in the kernel. In this case, the operations are the various machine-specific functions that tend to require a discussion with the hypervisor. They include things like disabling interrupts, changing processor control registers, changing memory mappings, etc. 

As an example, one of the members of paravirt_ops is: 

```
void (fastcall *irq_disable)(void);
```

The patch also defines a little function for use by the kernel: 

```
static inline void raw_local_irq_disable(void)
        {
        	paravirt_ops.irq_disable();
        }
```

As long as the kernel always uses this function to disable interrupts, it will use whatever implementation has been provided by the hypervisor which fills in paravirt_ops. 

The patch includes a set of operations for native (non-virtualized systems) which causes the kernel to behave as it did before - or which will bring this about, once the remaining bugs are fixed. That kernel may be a little slower, however, since many operations which were performed by in-line assembly code are now, instead, done through an indirect function call. To mitigate the worst performance impacts, the paravirt_ops patch set includes a self-patching mechanism to fix up some of the function calls - the interrupt-related ones, in particular. 

This interface may look a lot like VMI; both interfaces allow the replacement of important low-level operations with hypervisor-specific versions. The difference is that paravirt_ops is an inherently source-based interface, with no binary interface guarantees. It is assumed that this interface will change over time, as most other internal kernel interfaces do. In fact, since this is a relatively new area for kernel support, chances are that paravirt_ops will be more than usually volatile for some time. There is also, currently, no provision for loading the operations at run time, so kernels must be built to work with a specific hypervisor. 

On the surface, paravirt_ops thus looks like a competitor to VMI - a choice of open, mutable kernel interfaces against binary blobs and a fixed ABI. As it happens, however, there is a diverse set of developers working on paravirt_ops, including representatives from Xen and, yes, VMWare. Some of the VMI code has found its way into the initial paravirt_ops posting. All of the large players appear to be behind this development - a fact which will greatly ease its path into the kernel. 

So why are the VMWare developers still pushing for a binary interface? It would appear that they are considering the creation of a glue layer connecting paravirt_ops with the VMI binary interface. This design leaves the VMI people solely responsible for maintaining their ABI while freeing the kernel developers to mess with paravirt_ops at will. Some of the relevant developers feel more at ease with the VMI interface when it is connected this way, though there is some residual discomfort about the possibility of linking non-GPL binary hypervisor modules into the kernel. 

The paravirt_ops developers would like to get their code into the 2.6.19 kernel. That schedule looks ambitious, given that the merge window is due to open in a few weeks and that, as of this writing, paravirt_ops has not yet done any time in the -mm kernel. It is, however, an option which should disappear entirely when configured out, so inclusion in 2.6.19 might not be entirely out of the question.
