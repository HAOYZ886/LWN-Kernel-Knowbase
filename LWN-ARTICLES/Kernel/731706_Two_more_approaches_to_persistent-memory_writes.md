---
title: "Two more approaches to persistent-memory writes"
url: https://lwn.net/Articles/731706/
date: "August 23, 2017"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet August 23, 2017"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 23, 2017

The persistent-memory arrays we're told we'll all be able to get someday promise high-speed, byte-addressable storage in massive quantities. The Linux kernel community has been working to support this technology fully for a few years now, but there is one problem lacking a proper solution: allowing direct writes to persistent memory that is managed by a filesystem. None of the proposed solutions have yet made it into the mainline, but that hasn't stopped developers from trying; now two new patch sets addressing this issue are under consideration. 

Normally, filesystems are in control of all I/O to the underlying storage media; they use that control to ensure that the filesystem structure is consistent at all times. Even when a file on a traditional storage device is mapped into a process's virtual address space, the filesystem manages the writeback of modified pages from the page cache to persistent storage. Directly mapped persistent memory bypasses the filesystem, though, leading to a number of potential problems including inconsistent metadata or data corruption and loss if the filesystem relocates the file being modified. Solving this problem requires getting the filesystem back into the loop just far enough to avoid confusion while keeping the performance enabled by direct access to the storage media. 

Proposed solutions have included a special ["I know what I'm doing" flag](<https://lwn.net/Articles/678011/>) and, more recently, [a new system call named `daxctl()`](<https://lwn.net/Articles/726481/>) to freeze the state of a file's metadata so that the data could be safely modified in place. None of them have proved fully satisfactory, though, sending developers back to their keyboards to come up with a new approach. 

#### Synchronous page faults

One new contender is the [synchronous page faults patch set](<https://lwn.net/Articles/731588/>) from Jan Kara. It follows the lead of some of the previous attempts by ensuring that any needed filesystem metadata writes are completed before a process is allowed to modify directly mapped data. A new flag, `MAP_SYNC`, is added to the `mmap()` system call to request the synchronous behavior; that means, in particular: 

The guarantee provided by this flag is: While a block is writeably mapped into page tables of this mapping, it is guaranteed to be visible in the file at that offset also after a crash. 

In other words, the filesystem will not silently relocate the block, and it will ensure that the file's metadata is in a consistent state so that the blocks in question will be present after a crash. This is done by ensuring that any needed metadata writes have been done before the process is allowed to write pages affected by that metadata. 

When a persistent-memory region is mapped using `MAP_SYNC`, the memory-management code will check to see whether there are metadata writes pending for the affected file. It will not actually flush those writes out, though. Instead, the pages are mapped read-only with a special flag, forcing a page fault when the process first attempts to perform a write to one of those pages. The fault handler will then flush out any dirty metadata synchronously, set the page permissions to allow the write, and return. At that point, the process can write the page safely, since all the necessary metadata changes have already made it to persistent storage. 

The result is a relatively simple mechanism that will perform far better than the currently available alternative — manually calling `fsync()` before each write to persistent memory. The potential downside is that any write operation can now create a flurry of I/O as the filesystem flushes out dirty metadata. That can cause the process to block in what was supposed to be a simple memory write, introducing latency that may be unexpected and unwanted. Fear of that latency has helped to drive the quest for alternatives. 

#### MAP_DIRECT

One such alternative is the [`MAP_DIRECT` patch set](<https://lwn.net/Articles/730991/>) from Dan Williams. It can be thought of as the current form of the `daxctl()` patch mentioned above, though that new system call is no longer a part of the proposal. Instead, we have, once again, a new `mmap()` flag, but the proposed semantics are rather different. This flag eliminates the potential for write-fault latency by "sealing" the state of the file at the time it is mapped. 

When a filesystem sees a map request with `MAP_DIRECT`, it should ensure that all metadata related to the area being mapped is consistent on the storage media before continuing. Once the mapping has been made, the filesystem must reject any operation that would force a metadata write affecting the portion of the file that has been mapped. Blocks cannot be moved, for example, unless the filesystem can magically perform the move in an atomic manner that does not risk data loss for a concurrent process writing to that block. Operations like truncating the file, breaking the sharing of extents in the file, or allocating blocks will fail. This extends to allocating blocks for the region that has been mapped; the application must thus ensure that all of the relevant blocks are allocated before creating the mapping. 

An important aspect of this "sealing" operation is that it is a part of the filesystem's runtime state; it is not stored on the media itself. So, if the system crashes, the file will not be sealed after the reboot. The seal is only there to support a specific mapping and will go away when the mapping itself is taken down. It's also worth noting that the filesystem implementation may choose to only seal the portion of the file that has been mapped, or it may seal the entire file. 

An application that uses `MAP_DIRECT` will want a clear indication from the kernel that the file has indeed been sealed. Unfortunately, `mmap()` is one of those system calls that does not check for unknown flags; one can pass `MAP_DIRECT` on any existing kernel and not get an error back. To get around this problem, the patch set adds a new `mmap3()` variant that _does_ fail on unknown flags. Internally, the patch set adds a `mmap_supported_mask` field to the `file_operations` structure so that each low-level implementation can specify which flags it is able to handle. Requiring applications to use a new version of `mmap()` is not pretty, but there is no other way to solve the problem without an ABI change. 

Use of `MAP_DIRECT` requires the `CAP_LINUX_IMMUTABLE` capability; without that restriction, it was feared, it might be possible to carry out a denial-of-service attack by sealing a file that some other process needs to be able to change. As a result, this feature is not available to most users, which rather limits its usefulness. In an attempt to improve the situation, the patch set also adds a new `fcntl()` operation called `F_MAP_DIRECT`. This operation, which is also subject to the capability check, sets a flag on an open file that causes subsequent `mmap()` operations to act as if `MAP_DIRECT` had been specified, but without the capability check. The idea is that a privileged process could open a file and set this flag, then pass the file descriptor to an unprivileged process that does the actual work with that file. 

One advantage to `MAP_DIRECT` is that it has applications beyond just allowing high-performance applications to write directly to storage. The sealing mechanism is close to what the kernel needs anyway for files used for swapping, so some improvements may be possible there. It also makes it possible to set up DMA I/O operations from user-space drivers, a feature that is attractive in the RDMA realm, at least. 

Comments on both patch sets have been relatively muted after the most recent posting. Each is probably getting close to a point where it could be considered for inclusion. What has not happened, though, is any sort of discussion on which of the two is the better approach, or whether they should be combined somehow. So, while the community may be getting closer to a solution for direct writes to persistent memory, it will probably be a little while yet before any solution makes it upstream.
