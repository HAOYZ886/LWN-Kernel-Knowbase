---
title: "Huge pages part 3: Administration"
url: https://lwn.net/Articles/376606/
date: "March 3, 2010"
category: "Huge pages; Memory management-Huge pages"
author: "March 3, 2010 This article was contributed by Mel Gorman"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

March 3, 2010

This article was contributed by Mel Gorman

[_Editor's note: this is the third part in Mel Gorman's series on the use of huge pages in Linux. For those who missed them, a look at[part 1](<http://lwn.net/Articles/374424/>) and [part 2](<http://lwn.net/Articles/375096/>) is recommended before diving into this installment._] 

In this chapter, the setup and the administration of huge pages within the system is addressed. Part 2 discussed the different interfaces between user and kernel space such as hugetlbfs and shared memory. For an application to use these interfaces, though, the system must first be properly configured. Use of hugetlbfs requires only that the filesystem must be mounted; shared memory needs additional tuning and huge pages must also be allocated. Huge pages can be statically allocated as part of a pool early in the lifetime of the system or the pool can be allowed to grow dynamically as required. Libhugetlbfs provides a hugeadm utility that removes much of the tedium involved in these tasks. 

#### 1 Identifying Available Page Sizes

Since kernel `2.6.27`, Linux has supported more than one huge page size if the underlying hardware does. There will be one directory per page size supported in `/sys/kernel/mm/hugepages` and the "default" huge page size will be stored in the `Hugepagesize` field in `/proc/meminfo`. 

The default huge page size can be important. While hugetlbfs can specify the page size at mount time, the same option is not available for shared memory or MAP_HUGETLB. This can be important when using 1G pages on AMD or 16G pages on Power 5+ and later. The default huge page size can be set either with the last `hugepagesz=` option on the kernel command line (see below) or explicitly with `default_hugepagesz=`. 

Libhugetlbfs provides two means of identifying the huge page sizes. The first is using the `pagesize` utility with the `-H` switch printing the available huge page sizes and `-a` showing all page sizes. The programming equivalent are the `gethugepagesizes()` and `getpagesizes()` calls. 

#### 2 Huge Page Pool

Due to the inability to swap huge pages, none are allocated by default, so a pool must be configured with either a static or a dynamic size. The static size is the number of huge pages that are pre-allocated and guaranteed to be available for use by applications. Where it is known in advance how many huge pages are required, the static size should be set. 

The size of the static pool may be set in a number of ways. First, it may be set at boot-time using the `hugepages=` kernel boot parameter. If there are multiple huge page sizes, the `hugepagesz=` parameter must be used and interleaved with `hugepages=` as described in `Documentation/kernel-parameters`. For example, Power 5+ and later support multiple page sizes including 64K and 16M; both could be configured with: 

```
hugepagesz=64k hugepages=128 hugepagesz=16M hugepages=4
```

Second, the default huge page pool size can be set with the `vm.nr_hugepages` sysctl, which, again, tunes the default huge page pool. Third, it may be set via sysfs by finding the appropriate `nr_hugepages` virtual file below `/sys/kernel/mm/hugepages`. 

Knowing the exact huge page requirements in advance may not be possible. For example, the huge page requirements may be expected to vary throughout the lifetime of the system. In this case, the maximum number of additional huge pages that should be allocated is specified with the `vm.nr_overcommit_hugepages`. When a huge page pool does not have sufficient pages to satisfy a request for huge pages, an attempt to allocate up to `nr_overcommit_hugepages` is made. If an allocation fails, the result will be that `mmap()` will fail to avoid page fault failures as described in [Huge Page Fault Behaviour](<https://lwn.net/Articles/374424/#hpfb>) in part 1\. 

It is easiest to tune the pools with `hugeadm`. The `--pool-pages-min` argument specifies the minimum number of huge pages that are guaranteed to be available. The `--pool-pages-max` argument specifies the maximum number of huge pages that will exist in the system, whether statically or dynamically allocated. The page size can be specified or it can be simply `DEFAULT`. The amount to allocate can be specified as either a number of huge pages or a size requirement. 

In the following example, run on an x86 machine, the 4M huge page pool is being tuned. As 4M also happens to be the default huge page size, it could also have been specified as `DEFAULT:32M` and `DEFAULT:64M` respectively. 

```
$ hugeadm --pool-pages-min 4M:32M
        $ hugeadm --pool-pages-max 4M:64M
        $ hugeadm --pool-list
              Size  Minimum  Current  Maximum  Default
           4194304        8        8       16        *
```

To confirm the huge page pools are tuned to the satisfaction of requirements, `hugeadm --pool-list` will report on the minimum, maximum and current usage of huge pages of each size supported by the system. 

#### 3 Mounting HugeTLBFS

To access the special filesystem described in [HugeTLBFS](<https://lwn.net/Articles/375096/#hugetlbfs>) in part 2, it must first be mounted. What may be less obvious is that this is required to benefit from the use of the allocation API, or to automatically back segments with huge pages (as also described in part 2). The default huge page size is used for the mount if the `pagesize=` is not used. The following mounts two filesystem instances with different page sizes as supported on Power 5+. 

```
$ mount -t hugetlbfs /mnt/hugetlbfs-default
      $ mount -t hugetlbfs /mnt/hugetlbfs-64k -o pagesize=64K
```

Ordinarily it would be the responsibility of the administrator to set the permissions on this filesystem appropriately. `hugeadm` provides a range of different options for creating mount points with different permissions. The list of options are as follows and are self-explanatory. 

`--create-mounts`
     Creates a mount point for each available huge page size on this system under `/var/lib/hugetlbfs`. 

`--create-user-mounts <user>`
     Creates a mount point for each available huge page size under `/var/lib/hugetlbfs/<user>` usable by user <user>. 

`--create-group-mounts <group>`
     Creates a mount point for each available huge page size under `/var/lib/hugetlbfs/<group>` usable by group <group>. 

`--create-global-mounts`
     Creates a mount point for each available huge page size under `/var/lib/hugetlbfs/global` usable by anyone. 

It is up to the discretion of the administrator whether to call `hugeadm` from a system initialization script or to create appropriate `fstab` entries. If it is unclear what mount points already exist, use `--list-all-mounts` to list all current `hugetlbfs` mounts and the options used. 

#### 3.1 Quotas

A little-used feature of `hugetlbfs` is quota support which limits the number of huge pages that a filesystem instance can use even if more huge pages are available in the system. The expected use case would be to limit the number of huge pages available to a user or group. While it is not currently supported by `hugeadm`, the quota can be set with the `size=` option at mount time. 

#### 4 Enabling Shared Memory Use

There are two tunables that are relevant to the use of huge pages with shared memory. The first is the sysctl `kernel.shmmax` kernel parameter configured permanently in `/etc/sysctl.conf` or temporarily in `/proc/sys/kernel/shmmax`. The second is the sysctl `vm.hugetlb_shm_group` which stores which group ID (GID) is allowed to create shared memory segments. For example, lets say a JVM was to use shared memory with huge pages and ran as the user JVM with UID 1500 and GID 3000, then the value of this tunable should be 3000. 

Again, `hugeadm` is able to tune both of these parameters with the switches `--set-recommended-shmmax` and `--set-shm-group`. As the recommended value is calculated based on the size of the static and dynamic huge page pools, this should be called after the pools have been configured. 

#### 5 Huge Page Allocation Success Rates

If the huge page pool is statically allocated at boot-time, then this section will not be relevant as the huge pages are guaranteed to exist. In the event the system needs to dynamically allocate huge pages throughout its lifetime, then external fragmentation may be a problem. "External fragmentation" in this context refers to the inability of the system to allocate a huge page even if enough memory is free overall because the free memory is not physically contiguous. There are two means by which external fragmentation can be controlled, greatly increasing the success rate of huge page allocations. 

The first means is by tuning `vm.min_free_kbytes` to a higher value which helps the kernel's fragmentation-avoidance mechanism. The exact value depends on the type of system, the number of NUMA nodes and the huge page size, but `hugeadm` can calculate and set it with the `--set-recommended-min_free_kbytes` switch. If necessary, the effectiveness of this step can be measured by using the `trace_mm_page_alloc_extfrag` tracepoint and `ftrace` although how to do it is beyond the scope of this article. 

While the static huge page pool is guaranteed to be available as it has already been allocated, tuning `min_free_kbytes` improves the success rate when dynamically growing the huge page pool beyond its minimum size. The static pool sets the lower bound but there is no guaranteed upper bound on the number of huge pages that are available. For example, an administrator might request a minimum pool of 1G and a maximum pool 8G but fragmentation may mean that the real upper bound is 4G. 

If a guaranteed upper bound is required, a memory partition can be created using either the `kernelcore=` or `movablecore=` switch at boot time. These switches create a "Movable" zone that can be seen in `/proc/zoneinfo` or `/proc/buddyinfo`. Only pages that the kernel can migrate or reclaim exist in this zone. By default, huge pages are not allocated from this zone but it can be enabled by setting either `vm.hugepages_treat_as_movable` or using the `hugeadm` `--enable-zone-movable` switch. 

#### 6 Summary

In this chapter, four sets of system tunables were described. These relate to the allocation of huge pages, use of hugetlbfs filesystems, the use of shared memory, and simplifying the allocation of huge pages when dynamic pool sizing is in use. Once the administrator has made a choice, it should be implemented as part of a system initialization script. In the next chapter, it will be shown how some common benchmarks can be easily converted to use huge pages.
