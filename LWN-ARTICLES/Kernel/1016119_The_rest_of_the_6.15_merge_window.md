---
title: The rest of the 6.15 merge window
url: https://lwn.net/Articles/1016119/
date: "April 7, 2025"
category: "Releases-6.15"
author: "By Jonathan Corbet April 7, 2025"
---

By **Jonathan Corbet**  
April 7, 2025

Linus Torvalds [released 6.15-rc1](<https://lwn.net/Articles/1016576/>) and closed the 6.15 merge window on April 6\. By that time, 12,633 non-merge changesets had found their way into his repository; that is substantially more than were merged during the entire 6.14 development cycle. Just under 6,000 of those changesets were merged after [the first-half merge-window summary](<https://lwn.net/Articles/1015414/>) was written. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Some of the most interesting changes from the second half of the 6.15 merge window are: 

#### Architecture-specific

  * The RISC-V architecture has gained support for the [BFloat16](<https://en.wikipedia.org/wiki/Bfloat16_floating-point_format>) floating-point extension, the [Zaamo and Zalrsc](<https://lists.riscv.org/g/tech-announce/topic/public_review_for_zaamo_and/104583551>) extensions, and the [ZBKB](<https://github.com/riscv/riscv-bitmanip/blob/main/bitmanip/zbkb.adoc>) extension. 

#### Core kernel

  * The function and function-graph tracers have gained the ability to record the arguments to called functions; those arguments can then be examined in the trace output. 
  * The io_uring subsystem now supports zero-copy reception of network data, with eventual plans for allowing reception directly into device memory. See [this commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d9ac1d5fc951>) for documentation. 
  * It is also now possible to read epoll events via an io_uring operation. From [the merge message:](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6df9d086ffcb>)

> While this may seem counter-intuitive (and/or productive), the reasoning here is that quite a few existing epoll event loops can easily do a partial conversion to a completion based model, but are still stuck with one (or few) event types that remain readiness based. 
> 
> For that case, they then need to add the io_uring fd to the epoll context, and continue to rely on epoll_wait(2) for waiting on events. This misses out on the finer grained waiting that io_uring can do, to reduce context switches and wait for multiple events in one batch reliably. 
> 
> With adding support for reaping epoll events via io_uring, the whole legacy readiness based event types can still be reaped via epoll, with the overall waiting in the loop be driven by io_uring. 

  * The BPF subsystem has gained improved verification of programs with loops, some new [load-acquire and store-release instructions](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=880442305a39>), the ability to change extended attributes on files within BPF programs, and [a timed `may_goto` instruction](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3a6fa573c50f>). 
  * Also new in BPF is a "resilient queued spinlock" locking primitive. Its main purpose is to detect deadlock conditions at run time, enabling the loading of programs for which the verifier is unable to prove locking correctness. See [this merge message](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6ffb9017e932>) for an overview of these locks, and [this white paper](<https://github.com/kkdwivedi/rqspinlock/blob/main/rqspinlock.pdf>) for details. 
  * The [`try_alloc_pages()` allocation function](<https://lwn.net/Articles/1014220/#bpf>), which allows for memory allocation in any context (with a relatively high chance of failure) has been merged. It is intended to support BPF programs that may be running in highly restricted contexts. (See also: [this article](<https://lwn.net/Articles/1016001/>) from LSFMM+BPF 2025). 
  * It is now possible to check for the existence of guard pages in `/proc`; this addresses a regression (of sorts) experienced in some systems. It is also now possible to place guard pages in file-backed memory. See [this article](<https://lwn.net/Articles/1011366/>) for information about both changes. 
  * The tracking of mapping counts for large folios [has been significantly reworked](<https://lwn.net/Articles/1013649/>). The end result should be better tracking overall (and hopefully fewer security problems), but possibly slightly fuzzier working-set statistics. 
  * The page allocator has [seen some significant changes](<https://lwn.net/Articles/1014220/>) that are intended to greatly increase the reliability of huge-page allocations. 

#### Filesystems and block I/O

  * The filesystems in user space (FUSE) subsystem can now enforce timeouts on requests, enabling recovery when the user-space server becomes unresponsive. [This commit](<https://git.kernel.org/linus/9b17cb59a7db>) describes the sysctl knobs that control this feature. FUSE can also now handle file names longer than 1,024 characters. 

#### Hardware support

  * **Clock** : Rockchip RK3528 and RK3562 clock controllers, Allwinner A523/T527 clock-control units, and Qualcomm IPQ9574 NSS clock controllers. 
  * **GPIO and pin control** : Sophgo SG2042 and SG2044 pin controllers, Amlogic SoC pin controllers, AMD isp41-based pin controllers, and Allwinner A523 pin controllers. 
  * **Graphics** : Raydium RM67200-based DSI panels, Visionox RM692E5 display panels, Apple Summit display panels, Apple touch bars, and pre-DCP Apple display controllers. The "nova-core" stub driver has also been merged; this is the framework for what will eventually be the [nova](<https://lwn.net/Articles/990736/>) driver for NVIDIA GPUs. 
  * **Industrial I/O** : Silicon Labs Si7210 Hall-effect sensors, Broadcom APDS9160 ambient light and proximity sensors, Analog Device AD4851 data-acquisition systems, Analog Devices AD4030 and AD7191 analog-to-digital converters, Texas Instruments ADS7128 and ADS7138 analog-to-digital converters, Analog Devices ADIS16550 inertial sensors, and Dyna Image AL3000a ambient light sensors. 
  * **Input** : Apple Z2 touchscreens. 
  * **Miscellaneous** : AMD MDB Versal2 PCIe controllers, Broadcom BCM2712 MSI-X interrupt peripherals, Inside Secure SafeXcel EIP-93 crypto engines, Rockchip RK3588 random-number generators, Maxim MAX77705 battery chargers, Maxim MAX77705 power-management ICs, Maxim MAX77705 LED controllers, Samsung S2MPU05 regulators, Apple DWI 2-Wire interface backlight controllers, Intel timed IO PPS generators, CoreSight TMC control units, Qualcomm PCIe UNIPHY 28LP PHYs, Rockchip Samsung MIPI DCPHYs, SpacemiT K1 I2C adapters, Lenovo SE30 watchdog timers, and National Instruments 16500 UARTs. 
  * **USB** : Parade PS883x Type-C retimers. 

#### Miscellaneous

  * The "fwctl" subsystem, designed to pass command data directly through to complex firmware systems, has been merged. This subsystem [has been controversial](<https://lwn.net/Articles/969383/>) in the past, but [a 2024 Maintainers Summit session](<https://lwn.net/Articles/990802/>) came to the conclusion that it should be merged. There are three drivers included with this merge, one for CXL devices, one for mlx5 adapters, and one for AMD/Pensando distributed services cards. There is [some documentation](<https://origin.kernel.org/doc/html/next/userspace-api/fwctl/>) included with this new subsystem. 
  * The perf subsystem has gained the ability to perform latency profiling using scheduler information. See [this commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=802f0d58d52e>) for details, and for information on other perf enhancements added this time around. 

#### Security-related

  * The [Landlock](<https://docs.kernel.org/userspace-api/landlock.html>) security module has a new auditing mechanism that is intended to make it easier to understand access denials. [This commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8e2dd47b10e7>) contains documentation. 
  * The kernel may now optionally [seal a number of memory mappings](<https://lwn.net/Articles/1006375/>) against changes as a hardening measure. This feature was somewhat controversial because it is likely to break some applications, so it is disabled by default. Whether any distributors will dare to enable it remains to be seen. See [this documentation commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a8c15bb4008c>) for some more information. 

#### Internal kernel changes

  * The [runtime verification subsystem](<https://lwn.net/Articles/857862/>) has a new feature called "scheduler specification monitors" that allows multiple monitors to run concurrently and interact with each other. [This commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=03abeaa63c08>) contains documentation for this new feature. 
  * The new `traceoff_after_boot` command-line parameter will cause tracing to be disabled once the kernel has booted and started the `init` process. It is intended to help with the tracing of boot-related problems, ensuring that the trace data is not overwritten by the time a human is able to look at it. 
  * There is now support for running unit tests within Rust code with the new `#[kunit_tests()]` macro. 
  * New Rust abstractions cover high-resolution timers and coherent DMA mapping, which was the focus of [some disagreement](<https://lwn.net/Articles/1006805/>) earlier this year. 
  * There were 451 exported symbols removed and 447 added, for a net reduction of four. There are also twelve new kfuncs in 6.15. See [this page](<https://lwn.net/Articles/1016594/>) for the full list of additions and removals. 

This kernel now goes into the stabilization period, with the final 6.15 release happening on May 25 or June 1.
