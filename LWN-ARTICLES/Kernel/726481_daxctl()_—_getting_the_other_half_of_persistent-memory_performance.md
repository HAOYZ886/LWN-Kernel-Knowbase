---
title: "daxctl() — getting the other half of persistent-memory performance"
url: https://lwn.net/Articles/726481/
date: "June 26, 2017"
category: "DAX; Memory management-Nonvolatile memory"
author: "By Jonathan Corbet June 26, 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 26, 2017

Persistent memory promises high-speed, byte-addressable access to storage, with consequent benefits for all kinds of applications. But realizing those benefits has turned out to present a number of challenges for the Linux kernel community. Persistent memory is neither ordinary memory nor ordinary storage, so traditional approaches to memory and storage are not always well suited to this new world. A proposal for a new `daxctl()` system call, along with the ensuing discussion, shows how hard it can be to get the most out of persistent memory. 

The "DAX" mechanism allows an application to map a file in persistent-memory storage directly into its address space, bypassing the kernel's page cache. Thereafter, data in the file can be had via a pointer, with no need for I/O operations or copying the data through RAM. So far, so good, but there is a catch: this mode really only works for applications that are reading data from persistent memory. As soon as the time comes to do a write, things get more complicated. Writes can involve the allocation of blocks on the underlying storage device; they also create metadata updates that must be managed by the filesystem. If those metadata updates are not properly flushed out, the data cannot be considered properly written. 

The end result is that applications performing writes to persistent memory must call `fsync()` to be sure that those writes will not be lost. Even if the developer remembers to make those calls in all the right places, `fsync()` can create an arbitrary amount of I/O and, thus, impose arbitrary latencies on the calling application. Developers who go to the trouble of using DAX are doing so for performance reasons; such developers tend to respond to ideas like "arbitrary latencies" with poor humor at best. So they have been asking for a better solution. 

#### daxctl()

That is why Dan Williams wrote in the introduction to [this patch series](<https://lwn.net/Articles/725699/>) that ""the full promise of byte-addressable access to persistent memory has only been half realized via the filesystem-dax interface"". Realizing the other half requires getting the filesystem out of the loop when it comes to write access. If, say, a file could be set up so that no metadata changes would be needed in response to writes, the problem would simply go away. Applications would be able to write to DAX-mapped memory and, as long as they ensured that their own writes were flushed to persistent store (which can be done in user space with a couple of special instructions), there should be no concerns about lost metadata. 

Williams's proposal to implement this approach requires a couple of steps. The first is that the application needs to call [`fallocate()`](<http://man7.org/linux/man-pages/man2/fallocate.2.html>) to ensure that the file of interest actually has blocks allocated in persistent memory. Then it has to tell the kernel that the file is to be accessed via DAX and that the existing block allocations cannot be changed under any circumstances. That is done with a new system call: 

```
int daxctl(char *path, int flags, int align);
```

Here, `path` indicates the file of interest, `flags` indicates the desired action, and `align` is a hint regarding the size of pages that the application would like to use. The `DAXFILE_F_STATIC` flag, if present, will put the file into the "no changes allowed mode"; if the flag is absent, the file becomes an ordinary file once again. While the static mode is active, any operation on the file that would force metadata changes (changing its length with `truncate()`, for example) will fail with an error code. 

The implementation of this new mode would seem to require significant changes at the filesystem level, but it turns out that this functionality already exists. It is used by the swap subsystem which, when swapping to an ordinary file, needs to know where the blocks allocated to the file reside on disk. There are two pieces to this mechanism, the first of which is this `address_space_operations` method: 

```
/* Unfortunately this kludge is needed for FIBMAP. Don't use it */
        sector_t (*bmap)(struct address_space *s, sector_t sector);
```

A call to `bmap()` will return the physical block number on which the given `sector` is located; the swap subsystem uses this information to swap pages directly to the underlying device without involving the filesystem. To ensure that the list of physical blocks corresponding to the swap file does not change, the swap subsystem sets the `S_SWAPFILE` inode flag on the file. Tests sprinkled throughout the virtual filesystem layer (and the filesystems themselves) will block any operation that would change the layout of a file marked with this flag. 

This functionality is a close match to what DAX needs to make direct writes to persistent memory safe. So the `daxctl()` system call has simply repurposed this mechanism, putting the file into the no-metadata-changes mode while not actually swapping to it. 

#### MAP_SYNC

Christoph Hellwig was not slow to register his opposition to this idea. He would rather not see the `bmap()` method used anywhere else in the kernel; it is, in his opinion, [broken in a number of ways](<https://lwn.net/Articles/726489/>). Its use in swapping is also broken, he said, though ""we manage to paper over the fact"". He [suggested](<https://lwn.net/Articles/726490/>) that development should be focused instead on making DAX more stable before adding new features. 

An alternative approach, [proposed](<https://lwn.net/Articles/726492/>) by Andy Lutomirski, has been seen before: it was raised (under the name `MAP_SYNC`) during the ["I know what I'm doing" flag discussion](<https://lwn.net/Articles/678011/>) in early 2016. The core idea here is to get the filesystem to transparently ensure that any needed metadata changes are always in place before an application is allowed to write to a page affected by those changes. That would be done by write-protecting the affected pages, then flushing any needed changes as part of the process of handling a write fault on one of those pages. In theory, this approach would allow for a lot of use cases blocked by the `daxctl()` technique, including changing the length of files, copy-on-write semantics, concurrent access, and more. It's a seemingly simple idea that hides a lot of complexity; implementing it would not be trivial. 

Beyond implementation complexity, `MAP_SYNC` has another problem: it runs counter to the original low-latency goal. Flushing out the metadata changes to a filesystem can be a lengthy and complex task, requiring substantial amounts of CPU time and I/O. Putting that work into the page-fault handler means that page faults can take an arbitrarily long amount of time. As Dave Chinner [put it](<https://lwn.net/Articles/726494/>): 

Prediction for the MAP_SYNC future: frequent bug reports about huge, unpredictable page fault latencies on DAX files because every so often a page fault is required to sync tens of thousands of unrelated dirty objects because of filesystem journal ordering constraints. 

There was some discussion about how the impact of doing metadata updates in the page-fault handler could be reduced, but nobody has come forth with an idea that would reduce it to zero. Those ([such as Hellwig](<https://lwn.net/Articles/726495/>)) who support the `MAP_SYNC` approach acknowledge that cost, but see it as being preferable to adding a special-purpose interface that brings its own management difficulties. 

On the other hand, this work could lead to improvements to the swap subsystem as well, making it more robust and more compatible with filesystems (like Btrfs) whose copy-on-write semantics work poorly with the "no metadata changes" idea. There is another use case for this functionality: high-speed DMA directly to persistent memory also requires that the filesystem not make any unexpected changes to how the file is mapped. That, and the relative simplicity of Williams's patch, may help to push the `daxctl()` mechanism through, even though it is not universally popular. 

Arguably, the real lesson from this discussion is that persistent memory is not a perfect match to the semantics provided by the Unix API and current filesystems. It may eventually become clear that a different type of interface is needed, at least for applications that want to get maximum performance from this technology. Nobody really knows what that interface should look like yet, though, so the current approach of trying to retrofit new mechanisms onto what we have now would appear to be the best way forward.
