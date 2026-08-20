---
title: Fixing kmap_atomic()
url: https://lwn.net/Articles/356378/
date: "October 13, 2009"
category: "Memory management-Internal API; kmap atomic"
author: "By Jonathan Corbet October 13, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 13, 2009

Once upon a time, Linux was limited to less than 1GB of physical memory on 32-bit systems. This limit was imposed by two technical decisions: processes run with the same page tables in both kernel and user mode, and all physical memory had to be directly addressable by the kernel. Not changing page tables at every transition between kernel and user space is a significant performance win, but it forces the two modes to share the same 4GB address space. The directly-addressable requirement meant that total physical memory could not exceed the amount of virtual memory address space assigned to the kernel. Indeed, not even the full kernel space was available, due to the need to leave some space for I/O memory, `vmalloc()`, and so on. The normal split is 3GB for user space and 1GB for kernel space; that limited systems to a bit less than 1GB of physical memory. 

The way this problem was fixed was to create the concept of "high memory": memory which is not directly addressable by the kernel. Most of the time, the kernel does not need to directly manipulate much of the memory on the system; almost all user-space pages, for example, are usually only accessed in user mode. But, occasionally, the kernel must be able to reach into any page in the system. Zeroing new pages is one example; reading system call arguments from a user-space page is another. Since high-memory pages cannot live permanently in the kernel's virtual address space, the kernel needs a mechanism by which it can temporarily create a kernel-space address for specific high-memory pages. 

That mechanism is called `kmap()`; it takes a pointer to a `struct page` and returns a kernel-space virtual address for the page. When the kernel is done with the page, it must use `kunmap()` to unmap the page and make the address available for other mappings. `kmap()` works, but it can be slow; it requires translation lookaside buffer flushes and, potentially, cross-CPU interrupts for every mapping. Linus recently [commented](<https://lwn.net/Articles/356724/>) on the costs of high memory: 

HIGHMEM accesses really are very slow. You don't see that in user space, but I really have seen 25% performance differences between non-highmem builds and CONFIG_HIGHMEM4G enabled for things that try to put a lot of data in highmem (and the 64G one is even more expensive). And that was just with 2GB of RAM. 

All that costly work is done to keep the kernel-space mapping consistent across all processors in the system, even though many of these mappings are used only briefly, and only on a single CPU. 

To improve performance, the kernel developers introduced a special version: 

```
void *kmap_atomic(struct page *page, enum km_type idx);
```

Atomic kmap slots  
---  
`KM_BOUNCE_READ`  
`KM_SKB_SUNRPC_DATA`  
`KM_SKB_DATA_SOFTIRQ`  
`KM_USER0`  
`KM_USER1`  
`KM_BIO_SRC_IRQ`  
`KM_BIO_DST_IRQ`  
`KM_PTE0`  
`KM_PTE1`  
`KM_IRQ0`  
`KM_IRQ1`  
`KM_SOFTIRQ0`  
`KM_SOFTIRQ1`  
`KM_SYNC_ICACHE`  
`KM_SYNC_DCACHE`  
`KM_UML_USERCOPY`  
`KM_IRQ_PTE`  
`KM_NMI`  
`KM_NMI_PTE`  
This function differs from `kmap()` in some important ways. It only creates a mapping on the current CPU, so there is no need to bother other processors with it. It also creates the mapping using one of a very small set of kernel-space addresses. The caller must specify which address to use by way of the `idx` argument; these addresses are specified by a set of "slot" constants. For example, `KM_USER0` and `KM_USER1` are set aside for code called directly from user context \- system call implementations, generally. `KM_PTE0` is used for page table operations, `KM_SOFTIRQ0` is used in software interrupt mode, etc. There are about twenty of these slots defined in current kernels; see the list at the right for the 2.6.32 slots. 

The use of fixed slots requires that the code using these mappings be atomic - hence the name `kmap_atomic()`. If code holding an atomic kmap could be preempted, the thread which takes its place could use the same slots, with unfortunate results. The per-CPU nature of atomic mappings means that any cross-CPU migration would be disastrous. It's worth noting that there is no other protection against multiple use of specific slots; if two functions in a given call chain disagree about the use of `KM_USER0`, bad things are going to happen. In practice, this problem does not seem to actually bite people, though. 

This API has seen little change for years, but Peter Zijlstra has recently decided that it could use a face lift. The result is [a patch series](<http://lwn.net/Articles/356376/>) changing this fundamental interface and fixing the resulting compilation problems in over 200 files. The change is conceptually simple: the slots disappear, and the range of addresses is managed as a stack instead. After all, users of `kmap_atomic()` don't really care about which address they get; they just want an address that nobody else is using. The new API does force map and unmap operations to nest properly, but the atomic nature of these mappings means that usage generally fits that pattern anyway. 

There seems to be little question of this change being merged; Linus [welcomed it](<https://lwn.net/Articles/356386/>), saying ""I think this is how we should have done it originally."" There were some quibbles about the naming in the first version of the patch (`kmap_atomic()` had become `kmap_atomic_push()`), but that was easily fixed for the second iteration. 

It is also interesting to look at how this patch series was reworked. The first version was a single patch which did all of the changes at once. In response to reviewers, Peter broke the second version down into four steps: 

  1. Make sure that all atomic kmaps are created and destroyed in a strictly nested manner. There were a few places in the code where that did not happen; fixing it was usually just a matter of reordering a couple of `kunmap_atomic()` calls. 

  2. Switch to the stack-based mode without changing the `kmap_atomic()` prototype. So, after this patch, `kmap_atomic()` simply ignores the `idx` argument. 

  3. The `kmap_atomic()` prototype loses the `idx` argument; this is, by far, the largest patch of the series. 

  4. Various final details are fixed up. 

Doing things this way will make it a lot easier to debug any strange problems which result from the changes. The most significant change in terms of how the kernel works is step 2, so that's the patch which is most likely to create problems. But this organization makes that patch relatively small, so tracking down any residual bugs should be relatively easy. Instead, the really huge patch (part 3) should not really change the binary kernel at all, so the chances of it being problem-free are quite high. 

All that remains is getting this change merged. It's too late for 2.6.32, but putting it into linux-next is likely to create large numbers of patch conflicts. That is a common problem with wide-ranging patches like this, though; developers have gotten better over the years at maintaining them against a rapidly-changing kernel
