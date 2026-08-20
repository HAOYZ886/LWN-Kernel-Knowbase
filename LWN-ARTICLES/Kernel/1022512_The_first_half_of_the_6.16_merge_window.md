---
title: The first half of the 6.16 merge window
url: https://lwn.net/Articles/1022512/
date: "May 29, 2025"
category: "Releases-6.16"
author: "By Daroc Alden May 29, 2025"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Daroc Alden**  
May 29, 2025

As of this writing, 5,546 non-merge changesets have been pulled into the mainline kernel repository for the 6.16 release. This is a bit less than half of the total commits for 6.15, so the merge window is well on its way. Read on for our summary of the first half of the 6.16 merge window. 

As always, the [LWN kernel source database](<https://lwn.net/ksdb/>) provides summary statistics and historical breakdowns for subscribers. Here are the most interesting commits of the 6.16 merge window so far: 

#### Architecture-specific

  * [ Five-level page-table](<https://lwn.net/Articles/717293/>) support is now [ unconditionally enabled](<https://git.kernel.org/linus/7212b58d6d7133e4cd3c2295e1fb54febe284156>) for x86_64. ""Both Intel and AMD CPUs support 5-level paging, which is expected to become more widely adopted in the future. All major x86 Linux distributions have the feature enabled."" 
  * PowerPC now supports dynamic preemption (i.e., changing the kernel's [ preemption settings](<https://lwn.net/Articles/944686/>) at boot time). 
  * The intel_pstate driver now [ registers an energy model](<https://git.kernel.org/linus/7b010f9b906107ae4e5ac626329ab818b3f0a6b6>) for use with energy-aware scheduling on hybrid platforms without symmetric multithreading (also known as hyperthreading). 
  * Users can now [control C1 demotion](<https://git.kernel.org/linus/6138f3451516>) (the process whereby a CPU can independently decide to remain in a higher power state when the kernel tries to enter a lower one) with a sysfs knob, and [retrieve information](<https://git.kernel.org/linus/f1a50492f5bd>) on the capacity of different CPUs. 
  * [ Arm64 lazy-preemption support](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c8597e2dd8b6>) and [ scalable-matrix-extension support](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=33c4618d0ac0>) have both been merged. 

#### Core kernel

  * The kernel will [ now hand out pidfds](<https://git.kernel.org/linus/923ea4d4482be9475c31f1bc3691d7d74368348c>) for processes that have already exited when a SO_PEERPIDFD socket is used to request a pidfd for a thread group leader that has already been reaped. Since user-space code needs to handle processes dying while it holds a pidfd anyway, this should simplify error handling. 
  * Core dumps can [ now be sent](<https://git.kernel.org/linus/a3b4ca60f93ff3e8b41fffbf63bb02ef3b169c5e>) to an existing Unix socket, instead of being written to a file or spawning a user-mode helper. Christian Brauner [ hopes](<https://mastodon.social/@brauner/114592290899392625>) that this will reduce the number of CVEs related to the user-space core-dumping API. 
  * io_uring can now be used to [ create pipes](<https://git.kernel.org/linus/53db8a71ecb42c2ec5e9c6925269a750255f9af5>). 
  * Futexes are now [ NUMA](<https://git.kernel.org/linus/cec199c5e39bde7191a08087cc3d002ccfab31ff>) and [ mempolicy](<https://git.kernel.org/linus/c042c505210dc3453f378df432c10fff3d471bc5>) aware. This gives greater control over where futexes are placed in memory, so that they can be located close to the processes that will use them. 
  * There is now [ a command-line option](<https://git.kernel.org/linus/e34e0131fea1b0f63c2105a1958c94af2ee90f4d>) for enabling or disabling group scheduling of realtime tasks. 

#### Filesystems and block I/O

  * The bfs and omfs filesystems [ now use the new mount API](<https://git.kernel.org/linus/a1ae8ce78bb2600490d49a8f1ee88767b0e64381>). [ The API](<https://lwn.net/Articles/759499/>), added in 2019, has [slowly been adopted by](<https://lwn.net/Articles/979166/>) the kernel's many filesystems. 
  * A new sysctl knob, [ `vfs_cache_pressure_denom`](<https://git.kernel.org/linus/e7b9cea718eee4585a947b10086ca51ad27ef5d4>), indirectly controls the number of dentry cache entries ('dentries') that are preserved while the system is experiencing memory pressure. Specifically, the minimum proportion of dentries reclaimed during a memory reclamation event is 1/`vfs_cache_pressure_denom`, so setting a higher value will reduce the number of cache entries that are guaranteed to be collected, leaving more cache entries to be evicted or retained on their merits. 
  * Zoned loop block devices, which emulate a generic block device using multiple files on an existing file system, are now [ available](<https://git.kernel.org/linus/eb0570c7df23c2f32fe899fcdaf8fca9a5ecd51e>) and [ documented](<https://docs.kernel.org/next/admin-guide/blockdev/zoned_loop.html>). 
  * Among [ many bcachefs changes](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=522544fc71c27b4b432386c7919f71ecc79a3bfb>), the filesystem now supports only performing rebalance operations when the system has AC power. 
  * XFS [ supports atomic writes](<https://git.kernel.org/linus/6e7d71b3a0f9732863b6a1366c9d875cec52c842>). LWN [recently covered](<https://lwn.net/Articles/1016406/>) discussions about support for atomic writes in more filesystems. 
  * EROFS can now make use of Intel QAT hardware acceleration. 
  * ""Stupendous"" performance improvements on ext4 from [ a number of optimizations](<https://git.kernel.org/linus/d87d73895fcdbe6e45813efc473544433862364f>). 
  * The ancient [ `uselib()`](<https://man7.org/linux/man-pages/man2/uselib.2.html>) system call, which has been deprecated for some time, has now been [ removed](<https://git.kernel.org/linus/79beea2db0431536d79fc5d321225fb42f955466>), hopefully without breaking any user-space applications in the process. The system call is used to map dynamic libraries with writing disabled, so that they can be shared between different programs. That use case is served today by calling `mmap()` with appropriate flags. 
  * The block-layer maintainers have finally [ eliminated bounce buffering](<https://git.kernel.org/linus/eeadd68e2a5f6bfe0bf1038ec49e3a8d99eb5fe8>); see [this article](<https://lwn.net/Articles/1022655/>) for details. 

#### Hardware support

  * **Clock** : System Timer Modules on S32G NXP SoCs, EcoNet HPT timers, and Analog ADP5055 digital to analog converters. 
  * **GPIO and pin control** : EcoNet EN751221 SoCs, SG2044 SoCs, loongson, mc33xs2410 high-side switches, rzg2l-gpt pulse-width-modulation controllers, max77759 companion PMICs, VeriSilicon BLZP1600 GPIO interfaces, and Spacemit K1 SoCs. 
  * **Hardware monitoring** : PTC support on int340x, Airoha EN7581s, and IPQ5018 SoCs. 
  * **Input** : AMD HID2, Renesas RZ/G3Es, Rockchip RK3528s, and Samsung Exynos Autov920 processors. 
  * **Media** : MT8192 Spherion and MT8186 Corsola SoCs. 
  * **Miscellaneous** : SDHCI OF on the SpacemiT K1 SoCs. 
  * **Networking** : Qualcomm IPQ5018 WiFi chipsets. 
  * **Power** : Pegatron Chagall batteries, Maxim MAX8971 battery chargers, Huawei Matebook E Go chargers, Dimensity 1200 MT6893s, SM4450 power domains, RK3562 SoCs, Allwinner H6/H616 PRCM PPUs, and TI TPS65214 integrated power management chips. 
  * **Sound** : AMD ACP 7.x, Cirrus Logic CS35L63 amps and CS48L32 audio processors, Everest Semiconductor ES8375s and ES8389s, Longsoon-1 AC'97 audio codecs, NVIDIA Tegra264 SoCs, Richtek ALC203 and RT9123 codecs, Rockchip SAI controllers, Intel WCL, and DJM-V10 mixers. 

#### Miscellaneous

  * Filesystems and EFI variables [ can now be frozen](<https://git.kernel.org/linus/8dd53535f1e129b7d75c512dc271bff76461ab6b>) (and later unfrozen) on a best-effort basis during suspend and hibernate operations. 

#### Virtualization and containers

  * Control group shared tracking for recursive statistics turned out not to scale well to large numbers of control groups and [has been removed](<https://git.kernel.org/linus/3b66e6b3c098>). 
  * Virtual machines (VMs) can now [communicate](<https://git.kernel.org/linus/dd3922cf9d4d>) with a TPM device emulated by a Secure VM Service Module. 

#### Internal kernel changes

  * The timer API has undergone some [ significant renaming](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5e8bbb2caa4e>). Functions with irregular names have been converted to use the `timer_` prefix. For example, `init_timer_key()` is now `timer_init_key()`. There is another large refactoring expected near the end of the merge window. 
  * Rust modules can now use [ configfs](<https://git.kernel.org/linus/446cafc295bfc0e89da94a482fe8290bd8b429fb>). 
  * [ Stub drivers](<https://git.kernel.org/linus/b08494a8f7416e5f09907318c5460ad6f6e2a548>) for the nova DRM driver continue to make their way upstream. 
  * The virtual filesystem (VFS) interface has been [ cleaned up](<https://git.kernel.org/linus/6d5b940e1e14fcc20b5a3536647fe3c41b07d4f5>) to reduce the proliferation of confusing names. 
  * The `writepage()` method has been [ completely removed](<https://git.kernel.org/linus/dc762851444b32057709cb40e7cdb3054e60b646>) from `struct address_space_operations`, along with its remaining uses. 
  * The resctrl filesystem interface has now been [moved](<https://git.kernel.org/linus/7168ae330e81>) to its own directory, as the next step toward letting it be used on multiple architectures. 
  * The kernel-doc script, the origins of which predate the Git era, is used to extract documentation from the kernel source during the documentation-build process. Prior to 6.16, it was a horrifying Perl script full of impenetrable regular expressions. That script has been [ replaced](<https://git.kernel.org/linus/9f488ccd0f56>) with a Python version that is better integrated into the Sphinx build system. The regular expressions are no more penetrable than before, but the script as a whole will be far more maintainable. 
  * The kernel's build scripts had a separate option to handle specifying compiler flags that disable warnings, due to some inflexibility in the option-parsing code. The `cc-disable-warning` option is [no longer required](<https://git.kernel.org/linus/550ccb178de2>); it can be replaced by the normal `cc-option`. 
  * There is [a new DMA mapping API](<https://git.kernel.org/linus/23022f545610>), intended to provide an alternative to scatterlists. While it has taken some time to be finalized, the new API [ should provide better performance](<https://lwn.net/Articles/997563/>) for some high-bandwidth DMA devices. 
  * The idle-CPU-selection logic for sched_ext [can now apply topology-based optimizations](<https://git.kernel.org/linus/29f512f555ec>) in more cases, including to tasks with CPU affinities. 

There are 5,379 non-merge commits currently waiting in linux-next, so there are certainly more new kernel features to come. The merge window is expected to close on June 8. As usual, we will post another summary of those once the merge window closes.
