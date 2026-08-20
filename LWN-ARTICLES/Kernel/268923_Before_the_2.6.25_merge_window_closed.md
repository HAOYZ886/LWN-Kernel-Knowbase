---
title: Before the 2.6.25 merge window closed...
url: https://lwn.net/Articles/268923/
date: "February 12, 2008"
category: "Releases-2.6.25"
author: "By Jonathan Corbet February 12, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 12, 2008

The 2.6.25 merge window closed on February 10, after the merging of an eye-opening 9450 non-merge changesets. Most of the changes merged for 2.6.25 were covered in the [first](<http://lwn.net/Articles/266721/>) and [second](<http://lwn.net/Articles/267849/>) "what got merged" articles. This, the third in the series, covers the final 1900 patches merged before the window closed. 

User-visible changes include: 

  * There are new drivers for SC2681/SC2691-based serial ports, Dallas DS1511 timekeeping chips, AT91sam9 realtime clock devices, Compaq ASIC3 multi-function chips, Cell Broadband Engine memory controllers, Marvell MV64x60 memory controllers, PA Semi PWRficient NAND flash interfaces, Marvell Orion NAND flash controllers, Freescale eLBC NAND flash controllers, Sharp Zaurus SL-6000x keyboards, Fujitsu Lifebook Application Panel buttons, IPWireless 3G UMTS PCMCIA cards, intelligent storage device enclosures, Winbond W83L786NG and W83L786NR sensor chips, Texas Instruments ADS7828 12-bit 8-channel ADC devices, and Sony MemoryStick cards. 

  * Also added are updated video drivers for Radeon R500 chipsets (2D acceleration is now supported) and Intel i915 chipsets (suspend and resume now work properly). 

  * Several more obsolete OSS audio drivers have been removed. The old mxser driver has also been removed in favor of mxser_new, now called simply "mxser." 

  * File descriptors returned by `inotify_init()` now support signal-based (using `SIGIO`) I/O. There is also a new notification event (`IN_ATTRIB`) sent when the link count of a watched file changes. 

  * The mac80211 (formerly Devicescape) wireless subsystem is no longer marked "experimental." 

  * The memory use controller for containers has been merged. This controller was described in [this LWN article](<http://lwn.net/Articles/243795/>), but the patch has evolved somewhat since then and the details have changed. Some documentation can be found in [Documentation/controllers/memory.txt](<https://lwn.net/Articles/268937/>). 

  * ACPI thermal regulation support has been added; see [Documentation/thermal/sysfs-api.txt](<https://lwn.net/Articles/268958/>) for details on how it works. The ACPI code also now supports the Windows Management Instrumentation interface, and uses that support to make recent Acer laptops work. 

  * ACPI now provides support for users who want to override their system's Differentiated System Description Table (DSDT). 

  * The XFS filesystem now supports the `fallocate()` system call. 

  * ATA-over-Ethernet (AoE) now properly supports devices with multiple network interfaces (and, thus, multiple paths to the host). 

  * Support for the [MN10300 architecture](<http://www.semicon.panasonic.co.jp/e-micom/catalog/mn10300.htm>) (little-endian mode only) has been added. 

  * Support for a.out binaries has been removed from the ELF loader. Pure a.out systems will still work, though. 

  * Disk I/O statistics (as seen in `/proc/diskstats` and under `/sys/block`) have been augmented with more information about request merging and I/O wait time. 

  * The S390 architecture now implements dynamic page tables - processes will use 2-, 3-, or 4-level page tables depending on the size of their address space. 

  * The ext4 "in development" flag has been added; mounting an ext4 filesystem will now require an explicit "I know this might explode" option. 

Changes visible to kernel developers include: 

  * Many `nopage()` methods have been replaced by the newer `fault()` API; the near-term plan is to remove `nopage()` altogether. See [this article](<http://lwn.net/Articles/242625/>) for a description of the new way of "page not present" handling. 

  * This cycle has also seen a bit of a reinvigoration of the long-stalled project to eliminate the big kernel lock. A number of BKL-removal patches have been merged, with more certainly to come. 

  * A generic resource counter mechanism was merged as part of the memory controller patch set; see `<linux/res_counter.h>` for the details. 

  * `reserve_bootmem()` has a new `flags` parameter. Most callers will set it to `BOOTMEM_DEFAULT`; the kdump code, though, uses `BOOTMEM_EXCLUSIVE` to ensure that it is the only one to touch the memory. 

  * Most architectures now have support for `cmpxchg64()` and `cmpxchg_local()`. 

  * There is a new set of string functions: 

```
extern int strict_strtoul(const char *string, unsigned int base, 
                                      unsigned long *result);
            extern int strict_strtol(const char *string, unsigned int base,
            	       		     long *result);
            extern int strict_strtoull(const char *string, unsigned int base,
                                       unsigned long long *result);
            extern int strict_strtoll(const char *string, unsigned int base,
                                      long long *result);
```

These functions convert the given strings to various forms of `long` values, but they will return an error status if the given `string` value, as a whole, does not represent a proper integer value. These functions are now used in the parsing of kernel parameters. 

At this point, the merging of features is done (though there has been a bit of pushing for one or two things to slip in) and the stabilization period begins. With luck, that process will go a little more quickly than it did with 2.6.24.
