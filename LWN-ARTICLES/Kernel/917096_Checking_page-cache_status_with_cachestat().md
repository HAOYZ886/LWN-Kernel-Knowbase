---
title: "Checking page-cache status with cachestat()"
url: https://lwn.net/Articles/917096/
date: "December 6, 2022"
category: "Releases-6.5; System calls-cachestat"
author: "By Jonathan Corbet December 6, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 6, 2022

The kernel's page cache holds pages from files in RAM, allowing those pages to be accessed without expensive trips to persistent storage. Applications are normally entirely unaware of the page cache's operation; it speeds things up and that is all that matters. Some applications, though, can benefit from knowledge about how much of a given file is present in the page cache at any given time; the [proposed `cachestat()` system call](<https://lwn.net/ml/linux-kernel/20221205175140.1543229-1-nphamcs@gmail.com/>) from Nhat Pham is the latest in a long series of attempts to make that information available. 

In truth, even current kernels make it possible to learn which pages of a file are present in the page cache. The application just needs to map the file into its address space with [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>), after which a call to [`mincore()`](<https://man7.org/linux/man-pages/man2/mincore.2.html>) will return a vector showing which pages in that file are resident. This is an expensive solution, though; it requires setting up a (possibly unneeded otherwise) mapping and returns information that, for many applications, has a higher resolution than is necessary. 

The proposed `cachestat()` system call is rather simpler: 

```
struct cachestat {
            __u64 nr_cache;
            __u64 nr_dirty;
            __u64 nr_writeback;
            __u64 nr_evicted;
            __u64 nr_recently_evicted;
        };
    
        int cachestat(unsigned int fd, off_t offset, size_t len, size_t cstat_size, 
    		  struct cachestat *cstat);
```

This call will check the pages of the file indicated by `fd`, starting at the given `offset` and going for `len` bytes, and count the number of pages that are in various states of residency. The `offset` must be page-aligned; `len` will be rounded up to a multiple of the page size if needed. The counts will then be returned in the structure pointed to by `cstat`. In that structure, `nr_cache` is the number of pages in the given range that are present in the page cache, `nr_dirty` is the number of those pages that are dirty (have been modified and not yet written back to persistent storage), and `nr_writeback` is the number of pages currently being written back. 

The `nr_evicted` field provides the count of how many pages were once resident in the cache but have since been forced out, and `nr_recently_evicted` is the number of those that have been forced out in the recent past. In this case, the "recent past" is defined by the number of pages that have been evicted since the page in question was forced out; if that number is smaller than the process's working-set size, the eviction is deemed to be recent. These counts are obtained by looking at the shadow page-table information that was [added to the kernel](<https://lwn.net/Articles/495543/>) about ten years ago. 

The size of the `cachestat` structure must be provided to `cachestat()` as `cstat_size`. This interface allows new fields to be added to that structure in the future; if `cstat_size` is smaller than the size as known within the kernel, data will only be provided up to the provided size, preserving compatibility. (If, instead, `cstat_size` is larger than what the kernel expects, the call will fail with an `EINVAL` error). 

By not requiring the mapping and unmapping of the file(s) to be queried, `cachestat()` avoids most of the overhead created by the `mincore()` method. The fact that this call returns simple counts rather than detailed, by-page information is also helpful in the end; it seems that applications wanting this kind of information are interested in the number of cache-resident pages, but they don't really care about _which_ pages are resident. So there is no point in returning the more detailed data. 

One open question that is not well answered in this patch set, though, is: what kinds of applications will benefit from this information? When LWN [covered a similar effort](<https://lwn.net/Articles/371538/>) in 2010 (the system call was called `fincore()` then), the use case involved applications that call [`posix_fadvise()`](<https://man7.org/linux/man-pages/man2/posix_fadvise.2.html>) to bring data into the page cache prior to accessing it. These applications (SQLite is evidently one of them) know what their data-access patterns will be, but they have less information about how much of their data will fit into the page cache at any given time. By calling `cachestat()`, such an application can learn whether the pages it is prefetching into the cache are still there by the time it gets around to using them. If those pages are being evicted, the prefetching is overloading the page cache and causing more work overall; in such situations, the application can back off and get better performance. 

So `cachestat()` appears to be useful, but whether there is room in the kernel for this new system call remains to be seen. Attempts to add this functionality have faltered for over a decade, perhaps due to the highly specialized nature of the use case. But, just maybe, the new interface and renewed push for inclusion will get it over the bar this time.
