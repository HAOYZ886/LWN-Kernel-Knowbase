---
title: The state of guest_memfd
url: https://lwn.net/Articles/1016133/
date: "April 4, 2025"
category: "Confidential computing; Memory management-Address-space isolation"
author: "By Jonathan Corbet April 4, 2025 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 4, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

A typical cloud-computing host will share some of its memory with each guest that it runs. The host retains its access to that memory, though, meaning that it can readily dig through that memory in search of data that the guest would prefer to keep private. The [guest_memfd subsystem](<https://lwn.net/Articles/949277/>) removes (most of) the host's access to guest memory, making the guest's data more secure. In the memory-management track of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, David Hildenbrand ran a discussion on the state and future of this feature. 

Once upon a time, he said, virtual machines (VMs) were easy; they only had one type of memory provided by the host. In confidential-computing circles, this memory is deemed "shared", since it is accessible by both the guest and the host. More recently, the advent of confidential VMs has added the concept of private memory, which is only accessible to the guest. In hardware-backed implementations, an attempt to access guest-private memory by the host might even cause a machine check, stopping the show entirely. Private memory cannot (on the host) be mapped into user space, swapped out, or migrated. 

A participant asked whether DMA access by devices attached to the host was supported. In general, Hildenbrand said, that access is not allowed, but there is a "private device" concept that is being developed. Dan Williams said that the entire security model around private devices is that the device sets a ""trusted bit"" — to general laughter in the room. 

Hildenbrand said that private memory differs from the usual variety in numerous ways. It cannot be mapped, read from, or written to by the host, and often can be managed with small folios only. Jason Gunthorpe noted that every architecture implements private memory differently. Moving memory between the private and shared states, Hildenbrand said, can be challenging, and often can double the memory consumption of the guest. Conversion between types is usually done on individual 4KB base pages, splitting up huge pages. 

Current upstream work, he said, is aiming to integrate the concepts of both private and shared memory within guest_memfd; that would facilitate conversion between the two types. Fuad Tabba has been [doing some work](<https://lwn.net/ml/all/20250318161823.4005529-1-tabba@google.com/>) in this area. Getting there requires support for host-side memory mapping in guest_memfd; it would allow the host to easily access shared pages, but the host will still get a bus error if it attempts to access a private page. 

Converting pages from private to shared is always possible, Hildenbrand said, but the other way is harder. It is important to avoid having any private pages mapped into user space on the host, so the host must take pains to ensure that there are no unexpected references to any pages that are about to be made private. That is easy with small folios, because there is a single reference count to check, but the situation is more complicated with huge pages. 

There is some work underway, he said, by Ackerley Tng and Vishal Annapurve to improve huge-page support. It will allocate memory from the hugetlb subsystem, but then convert it into normal folios that can be mapped or split, if need be, to change smaller pieces between shared and private. Once the memory is freed, the huge folio is reconstructed and returned to hugetlb. 

Lorenzo Stoakes asked what the use case was for converting private memory back to shared; the answer is that VMs do need to make memory available to the hypervisor, often for device I/O. The discussion wandered for a while after that, including a suggestion that hugetlb should eventually be removed — a task that is [being worked on](<https://lwn.net/Articles/974491/>). 

In the absence of hardware support, guest_memfd can still support some device privacy by removing the private memory from the host's direct map, leaving the host with no way to address it. The result is not really confidential, but it provides some protection, Hildenbrand said. There is a problem, though: the memory-management developers do not want to expose the APIs that modify the direct map to loadable modules. But guest_memfd is implemented within KVM, which can be built as a loadable module. There were some suggestions of using the [restricted namespaces](<https://lwn.net/Articles/998221/>) feature to limit access to this API. Restricted namespaces have not yet found their way into the mainline, though. 

As the session ran out of time, Hildenbrand said that there would eventually need to be some sort of callback that could intercept folio-freeing operations. If the folio being freed has been shared out of a guest_memfd, the kernel will have to put it back where it came from, rather than making it generally available. This interception is currently done by testing for a specific folio subtype, but there are locking-related problems with that solution.
