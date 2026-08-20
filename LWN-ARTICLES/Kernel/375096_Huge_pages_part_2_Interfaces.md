---
title: "Huge pages part 2: Interfaces"
url: https://lwn.net/Articles/375096/
date: "February 24, 2010"
category: "Huge pages; Memory management-Huge pages"
author: "February 24, 2010 This article was contributed by Mel Gorman"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

February 24, 2010

This article was contributed by Mel Gorman

In an ideal world, the operating system would automatically use huge pages where appropriate, but there are a few problems. First, the operating system must decide when it is appropriate to promote base pages to huge pages requiring the maintenance of metadata which, itself, has an associated cost which may or may not be offset by the use of huge pages. Second, there can be architectural limitations that prevent a different page size being used within an address range once one page has been inserted. Finally, differences in TLB structure make predicting how many huge pages can be used and still be of benefit problematic. 

For these reasons, with one notable exception, operating systems provide a more explicit interface for huge pages to user space. It is up to application developers and system administrators to decide how they best be used. This chapter will cover the interfaces that exist for Linux. 

### 1 Shared Memory

One of the oldest interfaces backs shared memory segments created by `shmget()` with huge pages. Today, it is commonly used due to its simplicity and the length of time it has been supported. Huge pages are requested by specifying the `SHM_HUGETLB` flag and ensuring the `size` is huge-page-aligned. Examples of how to do this are included in the kernel source tree under [`Documentation/vm/hugetlbpage.txt`](<https://lwn.net/Articles/375098/>). 

A limitation of this interface is that only the default huge page size (as indicated by the `Hugepagesize` field in `/proc/meminfo`) will be used. If one wanted to use 16GB pages as supported on later versions of POWER for example, the `default_hugepagesz=` field must be used on the kernel command line as documented in `Documentation/kernel-parameters.txt` in the kernel source. 

The maximum amount of memory that can be committed to shared-memory huge pages is controlled by the `shmmax` sysctl parameter. This parameter will be discussed further in the next installment. 

### 2 HugeTLBFS

For the creation of shared or private mappings, Linux provides a RAM-based filesystem called "hugetlbfs." Every file on this filesystem is backed by huge pages and is accessed with `mmap()` or `read()`. If no options are specified at mount time, the default huge page size is used to back the files. To use a different page size, specify `pagesize=`. 

```
$ mount -t hugetlbfs none /mnt/hugetlbfs -o pagesize=64K
```

There are two ways to control the amount of memory which can be consumed by huge pages attached to a mount point. The `size=` mount parameter specifies (in bytes; the "K," "M," and "G" suffixes are understood) the maximum amount of memory which will be used by this mount. The size is rounded down to the nearest huge page size. It can also be specified as a percentage of the static huge page pool, though this option appears to be rarely used. The `nr_inodes=` parameter limits the number of files that can exist on the mount point which, in effect, limits the number of possible mappings. In combination, these options can be used to divvy up the available huge pages to groups or users in a shared system. 

Hugetlbfs is a bare interface to the huge page capabilities of the underlying hardware; taking advantage of it requires application awareness or library support. Libhugetlbfs makes heavy use of this interface when automatically backing regions with huge pages. 

For an application wishing to use the interface, the initial step is to discover the mount point by either reading `/proc/mounts` or using libhugetlbfs. Finding the mount point manually is relatively straightforward and already well known for `debugfs` but, for completeness, a very simple example program is shown below: 

> 

```
>     #include <stdio.h>
>     #include <string.h>
>     #include <stdlib.h>
>     #include <sys/param.h>
>     
>     char *find_hugetlbfs(char *fsmount, int len)
>     {
>     	char format[256];
>     	char fstype[256];
>     	char *ret = NULL;
>     	FILE *fd;
>     
>     	snprintf(format, 255, "%%*s %%%ds %%255s %%*s %%*d %%*d", len);
>     
>     	fd = fopen("/proc/mounts", "r");
>     	if (!fd) {
>     		perror("fopen");
>     		return NULL;
>     	}
>     
>     	while (fscanf(fd, format, fsmount, fstype) == 2) {
>     		if (!strcmp(fstype, "hugetlbfs")) {
>     			ret = fsmount;
>     			break;
>     		}
>     	}
>     
>     	fclose(fd);
>     	return ret;
>     }
>     
>     int main() {
>     	char buffer[PATH_MAX+1];
>     	printf("hugetlbfs mounted at %s\n", find_hugetlbfs(buffer, PATH_MAX));
>     	return 0;
>     }
>
```

When there are multiple mount points (to make different page sizes available), it gets more complicated; libhugetlbfs also provides a number of functions to help with these mount points. `hugetlbfs_find_path()` returns a mount point similar to the example program above, while `hugetlbfs_find_path_for_size()` will return a mount point for a specific huge page size. If the developer wishes to test a particular path to see if it `hugetlbfs` or not, use `hugetlbfs_test_path()`. 

### 3 Anonymous mmap()

As of kernel `2.6.32`, support is available that allows anonymous mappings to be created backed by huge pages with `mmap()` by specifying the flags `MAP_ANONYMOUS|MAP_HUGETLB`. These mappings can be private or shared. It is somewhat of an oversight that the amount of memory that can be pinned for anonymous `mmap()` is limited only by huge page availability. This potential problem may be addressed in future kernel releases. 

### 4 libhugetlbfs Allocation APIs

It is recognised that a number of applications want to simply get a buffer backed by huge pages. To facilitate this, libhugetlbfs provides two APIs since release 2.0, `get_hugepage_region()` and `get_huge_pages()` with corresponding free functions called `free_hugepage_region()` and `free_huge_pages()`. These are all provided with manual pages distributed with the libhugetlbfs package. 

`get_huge_pages()` is intended for use with the development of custom allocators and not as a drop-in replacement for `malloc()`. It is required that the size parameter to this API be hugepage-aligned which can be discovered with the function `gethugepagesize()`. 

If an application wants to allocate a number of very large buffers but is not concerned with alignment or some wastage, it should use `get_hugepage_region()`. The calling convention to this function is much more relaxed and will optionally fallback to using small pages if necessary. 

It is possible that applications need very tight control over how the mapping is placed in memory. If this is the case, libhugetlbfs provides `hugetlbfs_unlinked_fd()` and `hugetlbfs_unlinked_fd_for_size()` to create a file descriptor representing an unlinked file on a suitable `hugetlbfs` mount point. Using the file-descriptor, the application can `mmap()` with the appropriate parameters for accurate placement. 

Converting existing applications and libraries to use the API where applicable should be straightforward, but basic examples of how to do it with the [STREAM](<http://www.cs.virginia.edu/stream/>) memory bandwidth benchmark suite are available [gorman09a]. 

### 5 Automatic Backing of Memory Regions

While applications can be modified to use any of the interfaces, it imposes a significant burden on the application developer. To make life easier, libhugetlbfs can back a number of memory region types automatically when it is either pre-linked or pre-loaded. This process is described in the `HOWTO` documentation and manual pages that come with libhugetlbfs. 

Once loaded, libhugetlbfs's behaviour is determined by environment variables described in the `libhugetlbfs.7` manual page. As manipulating environment variables is time-consuming and error-prone, a `hugectl` utility exists that does much of the configuring automatically and will output what steps it took if the `--dry-run` switch is specified. 

To determine if huge pages are really being used, `/proc` can be examined, but libhugetlbfs will also warn if the verbosity is set sufficiently high and sufficient numbers of huge pages are not available. See below for an example of using a simple program that backs a 32MB segment with huge pages. Note how the first attempt to use huge pages failed and some configuration was required as no huge pages were previously configured on this system. 

The manual pages are quite comprehensive so this section will only give a brief introduction as to how different sections of memory can be backed by huge pages without modification. 

> 

```
>       $ ./hugetlbfs-shmget-test 
>       shmid: 0x2130007
>       shmaddr: 0xb5e37000
>       Starting the writes: ................................
>       Starting the Check...Done.
>     
>       $ hugectl --shm ./hugetlbfs-shmget-test
>       libhugetlbfs: WARNING: While overriding shmget(33554432) to add
>                              SHM_HUGETLB: Cannot allocate memory
>       libhugetlbfs: WARNING: Using small pages for shmget despite
>                              HUGETLB_SHM shmid: 0x2128007
>       shmaddr: 0xb5d57000
>       Starting the writes: ................................
>       Starting the Check...Done.
>     
>       $ hugeadm --pool-pages-min 4M:32M
>       $ hugectl --shm ./hugetlbfs-shmget-test 
>       shmid: 0x2158007
>       shmaddr: 0xb5c00000
>       Starting the writes: ................................
>       Starting the Check...Done.
>
```

#### 5.1 Shared Memory

When libhugetlbfs is preloaded or linked and the environment variable `HUGETLB_SHM` is set to `yes`, libhugetlbfs will override all calls to `shmget()`. Alternatively, launch the application with `hugectl $--$shm`. On setup, all `shmget()` requests will become aligned to a hugepage boundary and backed with huge pages if possible. If the system configuration does not allow huge pages to be used, the original request is honoured. 

#### 5.2 Heap

Glibc defines a `__morecore` hook that is is called when the heap size needs to be increased; libhugetlbfs uses this hook to create regions of memory backed by huge pages. Similar to shared memory, base pages are used when huge pages are not available. 

When libhugetlbfs is preloaded or linked and the environment variable `HUGETLB_MORECORE` set to `yes`, libhugetlbfs will configure the `__morecore` hook, causing `malloc()` requests will use huge pages. Alternatively, launch the application with `hugectl --heap`. 

Unlike shared memory, the page size can also be specified if more than one page size is supported by the system. The first example below uses the default page size (e.g. 16M on Power5+) and the second example explicitly overrides a default, using 64K pages. 

```
$ hugectl --heap ./target-application
        $ hugectl --heap=64k ./target-application
```

If the application has already been linked with libhugetlbfs, it may be necessary to specify `--no-preload` when using `--heap` so that an attempt is not made to load the library twice. 

By using the `__morecore` hook and setting the `mallopt()` option `M_MMAP_MAX` to zero, libhugetlbfs prevents glibc from making use of `brk()` to expand the heap. An application that calls `brk()` directly will be using base pages. 

If a custom memory allocator is being used, it must support the `__morecore` hook to use huge pages. An alternative may be to provide a wrapper around `malloc()` that called the real underlying `malloc()` or `get_hugepage_region()` depending on the size of the buffer. A heavy solution would be to provide a fully-fledged implementation of `malloc()` with libhugetlbfs that uses huge pages where appropriate, but this is currently unavailable due to the lack of a demonstrable use case. 

#### 5.3 Text and Data

Backing text or data is more involved as the application should first be relinked to align the sections to a huge page boundary. This is accomplished by linking against libhugetlbfs and specifying `-Wl,--hugetlbfs-align` \-- assuming the version of binutils installed is sufficiently recent. More information on relinking applications is described in the libhugetlbfs HOWTO. Once the application is relinked, as before control is with environment variables or with `hugectl`. 

```
$ hugectl --text --data --bss ./target-application
```

When backing text or data by text, the relevant sections are copied to files on the hugetlbfs filesystem and mapped with `mmap()`. The files are then unlinked so that the memory is freed on application exit. If the application is to be invoked multiple times, it is worth sharing that data by specifying the `--share-text` switch. The consequence is that the memory remains in use when the application exits and must be manually deleted. 

If it is not possible to relink the application, it is possible to force the loading of segments backed by huge pages by setting the environment variable `HUGETLB_FORCE_ELFMAP` to `yes`. This is not the preferred option as the method is not guaranteed to work. Segments must be large enough to overlap with a huge page and on architectures with limitations on where segments can be placed, it can be particularly problematic. 

#### 5.4 Stack

Currently, the stack cannot be backed by huge pages. Support was implemented in the past but the vast majority of applications did not aggressively use the stack. In many distributions, there are `ulimits` on the size of the stack that are smaller than a huge page size. Upon investigation, only the `bwaves` test from the SPEC CPU 2006 benchmark benefited from stacks being backed by huge pages and only then when using a commercial compiler. When compiled with `gcc`, there was no benefit, hence support was dropped. 

### 6 Summary

There are a small number of interfaces provided by Linux to access huge pages. While cumbersome to develop applications against, there is a programming API available with libhugetlbfs and it is possible to automatically back segments of memory with huge pages without application modification. In the next section, it will be discussed how the system should be tuned.
