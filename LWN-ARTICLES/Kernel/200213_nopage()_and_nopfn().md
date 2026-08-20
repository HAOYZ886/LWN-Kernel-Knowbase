---
title: nopage() and nopfn()
url: https://lwn.net/Articles/200213/
date: "September 20, 2006"
category: "Memory management-Internal API; nopfn"
author: 
---

The `nopage()` address space operation is charged with handling a major page fault within an address range. For address spaces backed by files, there is a generic `nopage()` method which causes the needed page to be read into memory. Device drivers also occasionally provide `nopage()` as part of their implementation of `mmap()`. In the driver case, a page fault is usually handled by finding the `struct page` corresponding to a memory-mapped buffer and passing that back to the kernel. 

There are a couple of errors which can be signaled by `nopage()`: `NOPAGE_SIGBUS` for truly bad addresses and `NOPAGE_OOM` for situations where an out-of-memory situation caused the attempt to handle the fault to fail. What is missing is the ability to indicate that `nopage()` was interrupted by a signal and the operation should be retried. That is not a situation which normally comes up in `nopage()` handlers which, if they must wait, usually do so in a non-interruptible manner. Benjamin Herrenschmidt has run into this issue, however, and has [proposed](<http://lwn.net/Articles/199642/>) a small change allowing a new `NOPAGE_RETRY` value. The response would be just as one would expect - the operation is retried later on, after the signal is handled. 

It turns out that Google [has a similar patch](<https://lwn.net/Articles/200215/>) which it applies internally, though the motivations are different. In Google's case, the patch exists to work around a performance problem that has been experienced there. This patch has not been submitted for merging because of [potential denial of service problems](<https://lwn.net/Articles/200216/>) and the fact that its author considers it to be a bit of a hack. 

Some form of this patch may well be merged eventually, but some more work seems called for first. The two patches make it clear that there are multiple reasons for returning `NOPAGE_RETRY`, so it might make sense to make that reason available to the higher levels of the page fault handler. That would allow some potential efficiency problems to be addressed, though the DOS scenario still presents potential problems. 

Meanwhile, one of the longstanding limitations of `nopage()` is that it can only handle situations where the relevant physical memory has a corresponding `struct page`. Those structures exist for main memory, but they do not exist when the memory is, for example, on a peripheral device and mapped into a PCI I/O memory region. Some architectures also do very strange things with special memory and multiple views of the same memory. In such cases, drivers must explicitly map the memory into user space with `remap_pfn_range()` instead of using `nopage()`. 

Jes Sorensen has, for some time, been carrying [a patch](<http://lwn.net/Articles/200156/>) which adds another address space operation called `nopfn()`. It is called in response to page faults only if there is no `nopage()` operation available; its job is to return a physical address (in the form of a page frame number) for the page which will satisfy the fault. That address will be stored directly into the process's page table, with no `struct page` required, and no reference counting performed. Jes has [an IA-64 special memory driver](<http://lwn.net/Articles/200199/>) which shows how this operation would be used. 

The idea has not been universally popular in the past - [Linus has opposed it](<http://www.gelato.unsw.edu.au/archives/linux-ia64/0603/17557.html>), as have others. To some it looks like a needless complication of the virtual memory subsystem; these people would rather see code use `remap_pfn_range()` or create special `page` structures as needed. There are a number of situations where the `nopfn()` is said to work better, however, and the pressures for its inclusion do not appear to be going away. So it will be interesting to see whether this one makes it into 2.6.19 or not.
