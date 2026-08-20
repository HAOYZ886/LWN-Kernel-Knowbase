---
title: Merged for 2.6.24
url: https://lwn.net/Articles/254559/
date: "October 17, 2007"
category: "Releases-2.6.24"
author: "By Jonathan Corbet October 17, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 17, 2007

The 2.6.24 merge window is open and, as of this writing, some 5,600 patches have found their way into the mainline. As usual, the list of changes is extensive. Some of the highlights among user-visible changes are: 

  * New drivers have been added for Toshiba TCM825x cameras (as found in the Nokia N800), Conexant cx23415 MPEG encoders (in framebuffer mode), Dual-DVB-T tuners, DiBcom DiB0070 tuners, Microtune MT2131 tuners, Samsung S5H1409 demodulators, Conexant CX23885/CX23887 PCIe bridge devices, Samsung LTV350QV LCD panel backlights, Kingsun KS-959 and "Dazzle" IrDA USB dongles, ADMtek ADM8211-based wireless network adapters, Marvell Libertas 8385 CF wireless adapters, Intel 82598-based 10GbE network cards, Intel PRO/Wireless 3945ABG/BG and Link AGN adapters (finally), IP1000 Gigabit Ethernet cards, ICH9 on-board Ethernet adapters, Tehuti Networks 10G Ethernet adapters, Sonics Silicon Backplane busses, several Ralink wireless adapters, "EMAC" built-in PowerPC ethernet controllers, Sun Neptune Ethernet adapters, Winchiphead CH341 USB-RS232 converters, Atmel AT32AP7000 USB device controllers, Blackfin bf548 ATAPI controllers, Atmel AVR32 parallel ATA controllers, National Semiconductor NS87415 parallel ATA controllers, Olympus MAUSB-10 and Fujifilm DPC-R1 flash card readers (which have the nice feature of allowing direct flash access without an intervening translation layer), TI DaVinci I2C controllers, Analog Devices ADT7470 temperature monitoring chips, IBM PowerExecutive power/temperature sensors, TI AR7 CPMAC Ethernet controllers, Siemens SX1 phones, Dallas Semiconductor DS1374 realtime clock chips, Atmel AT73C213 external sound devices, Cirrus Logic CS4270 codec devices, Gallant SC-6000 Audio Excel DSPs, and Atmel AT32AP and AT91 on-chip synchronous serial controllers. 

  * The Broadcom BCM43xx driver has been replaced by a new version which uses the mac80211 layer. Actually, there's two drivers: "b43" for newer adapters, and "b43legacy" for older 802.11b and 802.11g devices. 

  * The "dgrs" Digi RightSwitch driver has been removed from the kernel. This product, evidently, was never actually sold, so there should not be a whole lot of users inconvenienced by this change. 

  * The kernel now has basic support for SDIO peripherals. There is also now driver support for MMC/SD cards accessed via SPI controllers. 

  * The [CFS group scheduling](<http://lwn.net/Articles/240474/>) code has been merged. As of this writing, though, the feature cannot actually be turned on because the control groups code has not yet been merged. There is also a per-UID fair scheduling option which does work now. 

  * There is now support for RPC-based RDMA and the ability to mount NFS filesystems using RDMA. 

  * The traffic shaper, which limits bandwidth usage on network links, has been marked obsolete and scheduled for removal in 2.6.25. The much more flexible qdisc subsystem should be used instead. 

  * Allocation of UDP port numbers is now randomized. 

  * The netconsole code can now support multiple logging targets. 

  * Support for network namespaces has been added, enabling the virtualization of network-related resources in containers. Also merged is a virtual Ethernet driver which can be used to create network tunnels into (and out of) containers. 

  * The [Authenticated chunks](<http://tools.ietf.org/html/rfc4895>) protocol for the stream control transmission protocol (SCTP) is supported. 

  * A new "stateless NAT" implementation performs IPv4 network address translation in a much more resource-efficient manner. 

  * Support for the [SEED encryption algorithm](<http://www.ietf.org/rfc/rfc4269.txt>) has been added. 

  * Dynamic tick and clockevent support for the x86_64 architecture (now the 64-bit version of the new x86 architecture, see below) has been merged. 

  * Support for serial ATA port multipliers has been added. 

  * LZO compression is now supported in the JFFS2 filesystem. 

  * The [USB device authorization](<http://lwn.net/Articles/241980/>) code - a prerequisite to wireless USB support - has been merged. 

  * A new `hidraw` device provides access to a stream of unprocessed input device events for applications which have special needs in this area. 

  * The [per-device write throttling patches](<http://lwn.net/Articles/245600/>) have been merged; these patches should help the system keep heavy traffic on one block device from starving other devices. The [floating proportions patch](<http://lwn.net/Articles/245603/>), needed to support per-device throttling, has also gone in. 

  * There is a new sysctl flag for the out-of-memory killer (`oom_kill_allocating_task`). If this flag is set, the OOM killer will simply kill the process whose allocation brings about the out-of-memory situation instead of scanning through the system looking for better targets. 

  * Disk quota messages can now be delivered via a netlink socket. This should make it easier for graphical environments to inform the user when disk quota problems are encountered. 

  * The new `F_DUPFD_CLOEXEC` command causes `fcntl()` to duplicate a file descriptor and set the close-on-exec flag from the beginning. 

  * Block reservations have been added to the ext2 filesystem. 

  * The Linux security module interface is now [a non-module interface](<http://lwn.net/Articles/239962/>): the ability to load security modules on the fly has been removed. 

  * [File-based capability masks](<http://lwn.net/Articles/211883/>) are now supported. 

Important changes visible to kernel developers include: 

  * As expected, the [i386/x86_64 architecture merger](<http://lwn.net/Articles/243704/>) has happened. The result is a single architecture, called "x86," which can be built for 32-bit and 64-bit processors. 

  * The Video4Linux layer has some new internal support for composite devices involving more than one driver (many V4L2 devices involve, at a minimum, separate drivers for the controller and the sensor). 

  * Also in Video4Linux: the video-buf layer has been replaced with a more generic implementation which works with a wider range of devices (including USB devices and those which do not support scatter/gather DMA). 

  * The [large receive offload (LRO)](<http://lwn.net/Articles/243949/>) support layer has been merged into the networking subsystem. 

  * The NAPI interface used in network drivers [has been reworked](<http://lwn.net/Articles/244640/>) to better support devices with multiple transmit queues. 

  * The networking layer has a new function for printing MAC addresses: 

```
char *print_mac(char *buf, const u8 *addr);
```

The `buf` buffer should be declared with `DECLARE_MAC_BUF()`; the output is suitable for formatting in `printk()` with "`%s`". 

  * The `NETIF_F_LLTX` (lockless transmit) flag for network devices has been deprecated and should not be used in new code. 

  * The functions `ktime_sub_us()` and `ktime_sub_ns()` have been added; they subtract the given number of microseconds or nanoseconds from a `ktime_t` value. 

  * The `hard_header()` method has been removed from `struct net_device`; it has been replaced by a per-protocol `header_ops` structure pointer. 

  * The debugfs filesystem has some new functions (`debugfs_create_x8()`, `debugfs_create_x16()`, `debugfs_create_x32()`) which make it easy to export files containing hexadecimal numbers. 

  * Various small sysfs-related API changes have been made. The `name` field has been removed from the `kobject` structure. The prototypes of the user-event callbacks have been changed. Many of the subsystem-related calls have been removed - subsystems never really did much of anything anyway; `get_bus()` and `put_bus()` are also gone. 

  * A new value `DMA_MASK_NONE` can be stored in the `device` structure `dma_mask` field to indicate that the device is incapable of performing DMA. 

  * The VFS has a couple of new address space operations (`write_begin()` and `write_end()`) aimed at fixing some deadlock scenarios; see [this article](<https://lwn.net/Articles/254856/>) for more information. 

  * The [scatterlist chaining](<http://lwn.net/Articles/234617/>) patches have been merged and many parts of the kernel have been updated to use this feature. 

  * The `CFLAGS=` and `CPPFLAGS=` options now work with the kernel build system in the expected way: they add flags to be passed to the C compiler and preprocessor, respectively. 

  * The prototype for slab constructor callbacks has changed to: 

```
void (*ctor)(struct kmem_cache *cache, void *object);
```

The unused `flags` argument has been removed and the order of the other two arguments has been reversed to match other slab functions. 

  * The `DECLARE_MUTEX_LOCKED()` macro has been removed. 

  * The long-deprecated `SA_*` interrupt flags have been removed in favor of the `IRQF_*` equivalents. 

  * A number of block layer utilities have seen prototype changes. The most evident change, perhaps, is `bio_endio()` and the associated `bio_end_io_t` callback: 

```
void bio_endio(struct bio *bio, int error);
            typedef void (bio_end_io_t) (struct bio *, int);
```

These functions now always completes the entire BIO, so the `size` argument has been removed. 

As of this writing, the 2.6.24 merge window can be expected to remain open for up to another week. So expect more changes to go into the mainline before this development cycle goes into the stabilization phase.
