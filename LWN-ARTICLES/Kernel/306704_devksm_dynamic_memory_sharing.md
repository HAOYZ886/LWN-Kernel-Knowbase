---
title: "/dev/ksm: dynamic memory sharing"
url: https://lwn.net/Articles/306704/
date: "November 12, 2008"
category: "Memory management-Kernel samepage merging"
author: "By Jonathan Corbet November 12, 2008"
---

By **Jonathan Corbet**  
November 12, 2008

The kernel generally goes out of its way to share identical memory pages between processes. Program text is always shared, for example. But writable pages will also be shared between processes when the kernel knows that the contents of the memory are the same for all processes involved. When a process calls `fork()`, all writable pages are turned into copy-on-write (COW) pages and shared between the parent and child. As long as neither process modified the contents of any given page, that sharing can continue, with a corresponding reduction in memory use. 

Copy-on-write with `fork()` works because the kernel knows that each process expects to find the same contents in those pages. When the kernel lacks that knowledge, though, it will generally be unable to arrange sharing of identical pages. One might not think that this would ordinarily be a problem, but the KVM developers have come up with a couple of situations where this kind of sharing opportunity might come about. Your editor cannot resist [this case](<https://lwn.net/Articles/306706/>) proposed by Avi Kivity: 

Consider the typical multiuser gnome minicomputer with all 150 users reading lwn.net at the same time instead of working. You could share the firefox rendered page cache, reducing memory utilization drastically. 

Beyond such typical systems, though, consider the case of a host running a number of virtualized guests. Those guests will not share a process-tree relationship which makes the sharing of pages between them easy, but they may well be using a substantial portion of their memory to hold identical contents. If that host could find a way to force the sharing of pages with identical contents, it should be able to make much better use of its memory and, as a result, run more guests. This is the kind of thing which gets the attention of virtualization developers. So the hackers at ~~Qumranet~~ Red Hat (Izik Eidus, Andrea Arcanageli, and Chris Wright in particular) have put together a mechanism to make that kind of sharing happen. The resulting code, called KSM, was recently [posted](<http://lwn.net/Articles/306642/>) for wider review. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

KSM takes the form of a device driver for a single, virtual device: `/dev/ksm`. A process which wants to take part in the page sharing regime can open that device and register (with an `ioctl()` call) a portion of its address space with the KSM driver. Once the page sharing mechanism is turned on (via another `ioctl()`), the kernel will start looking for pages to share. 

The algorithm is relatively simple. The KSM driver, inside a kernel thread, picks one of the memory regions registered with it and start scanning over it. For each page which is resident in memory, KSM will generate an SHA1 hash of the page's contents. That hash will then be used to look up other pages with the same hash value. If a subsequent `memcmp()` call shows that the contents of the pages are truly identical, all processes with a reference to the scanned page will be pointed (in COW mode) to the other one, and the redundant page will be returned to the system. As long as nobody modifies the page, the sharing can continue; once a write operation happens, the page will be copied and the sharing will end. 

The kernel thread will scan up to a maximum number of pages before going to sleep for a while. Both the number of pages to scan and the sleep period are passed in as parameters to the `ioctl()` call which starts scanning. A user-space control process can also pause scanning via another `ioctl()` call. 

The initial [response](<https://lwn.net/Articles/306710/>) to the patch from Andrew Morton was not entirely enthusiastic: 

The whole approach seems wrong to me. The kernel lost track of these pages and then we run around post-facto trying to fix that up again. Please explain (for the changelog) why the kernel cannot get this right via the usual sharing, refcounting and COWing approaches. 

The [answer](<https://lwn.net/Articles/306711/>) from Avi Kivity was reasonably clear: 

For kvm, the kernel never knew those pages were shared. They are loaded from independent (possibly compressed and encrypted) disk images. These images are different; but some pages happen to be the same because they came from the same installation media. 

Izik Eidus [adds](<https://lwn.net/Articles/306713/>) that, with this patch, a host running a bunch of Windows guests is able to overcommit its memory 300% without terribly ill effects. This technique, it seems, is especially effective with Windows guests: Windows apparently zeroes all freed memory, so each guest's list of free pages can be coalesced down to a single, shared page full of zeroes. 

What has not been done (or, at least, not posted) is any sort of benchmarking of the impact KSM has on a running system. The scanning, hashing, and comparing of pages will require some CPU time, and it is likely to have noticeable cache effects as well. If you are trying to run dozens of Windows guests, cache effects may well be relatively low on your list of problems. But that cost may be sufficient to prevent the more general use of KSM, even though systems which are not using virtualization at all may still have a lot of pages with identical contents.
