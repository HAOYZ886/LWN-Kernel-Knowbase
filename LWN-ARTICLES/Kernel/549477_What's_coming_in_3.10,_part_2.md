---
title: "What's coming in 3.10, part 2"
url: https://lwn.net/Articles/549477/
date: "May 8, 2013"
category: "Releases-3.10"
author: "By Jonathan Corbet May 8, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 8, 2013

As of this writing, 10,892 non-merge changesets have been pulled into the mainline repository for the 3.10 development cycle, nearly 5,800 since [last week's summary](<https://lwn.net/Articles/548834/>). That does not yet make 3.10 the busiest merge window ever, but chances are good that the current record holder (3.8, at 10,901 changesets) will be surpassed in the very near future. In other words, there has been a lot going on. 

User-visible changes merged since last week's summary include: 

  * The "full dynamic tick" patch set has been merged, allowing non-idle CPUs to run without the periodic timer interrupt in some situations. See [this article](<https://lwn.net/Articles/549580/>) for more information about this feature. 

  * The TCP stack has gained support for the [tail loss probe algorithm](<http://tools.ietf.org/html/draft-dukkipati-tcpm-tcp-loss-probe-01>), which can speed recovery when a packet is dropped at the end of a connection. Many connections are short and latency-sensitive, so tail loss has a bigger effect than one might think. From [the commit](<http://git.kernel.org/linus/6ba8a3b19e764b6a65e4030ab0999be50c291e6c>): ""The TLP patch series have been extensively tested on Google Web servers. It is most effective for short Web transactions, where it reduced RTOs [retransmission timeouts] by 15% and improved HTTP response time (average by 6%, 99th percentile by 10%)."" 

  * The batman-adv mesh networking subsystem now supports "[network coding](<http://www.open-mesh.org/projects/batman-adv/wiki/NetworkCoding-technical>)," a mechanism that can, in some situations, transmit two packets to two destinations simultaneously by exclusive-ORing them together. 

  * The [TCP cookie transactions](<https://en.wikipedia.org/wiki/TCP_Cookie_Transactions>) mechanism has been removed. It was never fully implemented and had a performance cost. 

  * The IPv6 protocol is now supported over IEEE 1394 ("Firewire"). 

  * The netlink interface now supports memory-mapped I/O, eliminating the need to copy data between user and kernel space. Unsurprisingly, this involves the addition of yet another ring buffer implementation to the kernel. See [Documentation/networking/netlink_mmap.txt](<https://lwn.net/Articles/549487/>) for details. 

  * The "single zImage" goal for the ARM architecture is getting closer; several more platforms have gained multiplatform support. The biggest holdout at this point is Samsung's Exynos, which was reverted prior to the pull request because it turned out not to be quite ready. 

  * The XFS filesystem continues to gain checksums and other identifying information in all metadata blocks, aiding filesystem consistency checks and the cleanup process if something goes wrong. The new documentation file [Documentation/filesystems/xfs-self-describing-metadata.txt](<https://lwn.net/Articles/549496/>) has more information about this work and its goals. 

  * The ARM [multi-cluster power management](<https://lwn.net/Articles/539082/>) patch set has been merged, clearing the way for further big.LITTLE support, including the just-posted [in-kernel switcher code](<https://lwn.net/Articles/549473/>). 

  * The Ceph filesystem has new support for [RBD layering](<http://ceph.com/docs/master/dev/rbd-layering/>), essentially a mechanism for the quick creation of copy-on-write clones of filesystem snapshots. 

  * Various tweaks to the reader/writer semaphore code have led to [some impressive performance improvements](<https://lwn.net/Articles/549551/>) for some workloads. Pgbench results improved as much as 100% in one case. 

  * New hardware support includes: 

    * **Boards and processors** : Atmel SAMA5D3 processors, Freescale T4240 QDS and B4 QDS boards, Renesas R8A73A4, R8A7778, and R8A7790 processors, and Freescale i.MX6 DualLite/Solo processors. 

    * **Audio** : TI TAS5086 codecs and Asahi Kasei AK5386 codecs. 

    * **Input** : IMS passenger control units (in-seat controllers for airline entertainment systems). 

    * **Miscellaneous** : Samsung S3C64XX SoC pin controllers, VIA VT8500 and Wondermedia WM8xxx-series pinmux/gpio controllers, Freescale SAHARA crypto accelerators, Broadcom BCM2835 random number generators, ChromeOS embedded I2C, SPI, and keyboard controllers, Silicon Laboratories 4761/64/68 AM/FM radios, NVIDIA Tegra host1x DMA engines, Aeroflex Gaisler GRGPIO cores, TI LP5562 LED controllers, Samsung S3C, S5P and Exynos SoC PWM timers, Renesas R-Car GPIO controllers, and TI DA8xx/OMAP-L13x remote processors. 

    * **Networking** : Ralink RT5572 wireless network interfaces, Realtek RTL8188EE wireless network adapters, and Realtek RTL8152-based Ethernet adapters. 

    * **Virtualized** : CAIF link layer over virtio and QXL virtual GPUs.  

Changes visible to kernel developers include: 

  * NAPI-using network drivers that set themselves up with a weight greater than 64 will generate a user-visible warning. Larger weights can create unfairness on systems with multiple interfaces; they can also interfere with the proper functioning of the [byte queue limits](<https://lwn.net/Articles/454390/>) mechanism. 

  * The I2C layer has a new [bus arbitration module](<http://git.kernel.org/linus/b81dfaa01f7057dde4a6356740c023db4e2ce83b>) for multi-master situations and a new [bus recovery infrastructure](<http://git.kernel.org/linus/5f9296ba21b3c395e53dd84e7ff9578f97f24295>) as well. 

  * The ALSA ASoC subsystem has a new [generic dmaengine_pcm driver](<http://git.kernel.org/linus/28c4468b00a1e55e08cc20117de968f7c6275441>) that is able to replace a lot of platform-specific PCM implementations. 

  * Developers wanting to tweak the randomness of "randconfig" builds can do so with the new `KCONFIG_PROBABILITY` environment variable. 

  * The `release()` function in `struct block_device_operations` now returns `void`. 

  * There is a new "reset controller" driver class. Reset controllers are able to force-reset another system component by way of a mechanism like a GPIO line. 

A two-week merge window would close on May 13. Given the volume of code merged so far, though, it would not be at all surprising if Linus declared the end a little sooner than that.
