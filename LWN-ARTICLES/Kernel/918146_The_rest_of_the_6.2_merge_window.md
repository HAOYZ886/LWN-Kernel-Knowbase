---
title: The rest of the 6.2 merge window
url: https://lwn.net/Articles/918146/
date: "December 27, 2022"
category: "Releases-6.2"
author: "By Jonathan Corbet December 27, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 27, 2022

The world got a special Christmas present from Linus Torvalds this year in the form of the [6.2-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=wgf929uGOVpiWALPyC7pv_9KbwB2EAvQ3C4woshZZ5zqQ@mail.gmail.com/>) kernel prepatch. By the time the merge window closed, 13,687 non-merge changesets had been pulled into the mainline for the 6.2 release. This was the busiest merge window since 5.13 (which brought in 14.231 changesets) in mid-2021, and quite a bit busier than 6.1 was — but comparable to the late 5.x releases. Just under 4,000 of those changesets were pulled after [the first-half summary](<https://lwn.net/Articles/917733/>) was written; there were quite a few significant changes to be found in those late-arriving patches. 

The most significant changes pulled in the latter part of the 6.2 merge window include: 

#### Architecture-specific

  * The kernel can now perform [return stack buffer stuffing](<https://lwn.net/Articles/901834/>) to mitigate the Retbleed speculative-execution vulnerability on some Intel processor generations with a much lower performance cost. 
  * The x86 architecture has also gained support for a control-flow integrity mechanism called [FineIBT](<https://git.kernel.org/linus/931ab63664f0>). 
  * There is a new [qspinlock](<https://lwn.net/Articles/590243/>) implementation for the PowerPC architecture; it should provide improved performance and fix some lockup problems seen in extreme cases. 
  * LoongArch has gained support for ftrace, suspend, hibernation, and stack protection. 

#### Core kernel

  * The zram device can now recompress data streams to achieve better compression rates; see [this documentation commit](<https://git.kernel.org/linus/443dd798062c>) for details. 
  * Shared anonymous memory areas can now be named; this capability extends the current [memory-naming feature](<https://lwn.net/Articles/867818/>), which was previously limited to private memory. 
  * The new `trace_trigger=` command-line option can enable a tracing trigger at boot time. 

#### Filesystems and block I/O

  * There is a new set of sysfs knobs that can be used to fine-tune how much of the system's page cache can be used by pages destined to be written back to a specific device. See the documentation commits for [`strict_limit`](<https://git.kernel.org/linus/16b837eb84e6>), [`max_bytes`](<https://git.kernel.org/linus/c354d9268d78>), [`min_bytes`](<https://git.kernel.org/linus/9c832a8d5717>), [`max_ratio_fine`](<https://git.kernel.org/linus/54790f30fea7>), and [`min_ratio_fine`](<https://git.kernel.org/linus/eba39236f18d>) for details. 
  * The F2FS filesystem has gained an "atomic replace" `ioctl()` operation that can write data to a file and truncate it in a single atomic operation. F2FS has also gained a block-based extent cache that can be used to determine which data is hot (in active use) or cold; [this commit](<https://git.kernel.org/linus/71644dff4811>) contains a little information. 
  * The ntfs3 filesystem has a few new mount options, starting with the undocumented `nocase`, which appears to control case-sensitive lookups. The `windows_name` option will prevent the creation of file names that Windows would not allow, and `hide_dot_files` controls whether files whose names start with "`.`" are marked as being hidden. 

#### Hardware support

  * **Industrial I/O** : Analog Devices MAX11410 and AD4130 analog-to-digital converters, MediaTek MT6370 analog-to-digital converters, Kionix KX022A tri-axis digital accelerometers, Maxim MAX30208 digital temperature sensors, Analog Devices AD74115H I/O controllers, and Analog Devices ADF4377 microwave wideband synthesizers. 
  * **Miscellaneous** : Microsoft Azure network adapters, Baikal-T1 PCIe controllers, Ampere Computing SMPro error monitors, Lattice sysCONFIG SPI FPGA managers, Advantech embedded controller watchdog timers, Renesas R-Car S4-8 Ethernet SERDES PHYs, TI TPS65219 power management ICs, and Xilinx R5 remote processors. 
  * **Also** : "iommufd" is a new user-space API for the control of I/O memory-management units; see [Documentation/userspace-api/iommufd.rst](<https://docs.kernel.org/next/userspace-api/iommufd.html>) for details. 

#### Miscellaneous

  * As always, the perf tool has seen a lot of enhancements; see [this merge message](<https://git.kernel.org/linus/aa4800e31c54>) and [this one](<https://git.kernel.org/linus/d1ac1a2b1426>) (which drew [an unhappy response](<https://lwn.net/ml/linux-kernel/CAHk-=wi4u5rpaPrOokekEg_5Sud=8SZhA=KTALXNOC_S89jKBg@mail.gmail.com/>) from Torvalds) for details. 
  * The [copyleft-next license](<https://git.kernel.org/linus/ad9f64cd2d4a>) has been added as a valid license for kernel contributions — but only when dual-licensed with a GPL-compatible license. 

#### Security-related

  * The kernel can now place [an upper limit](<https://lwn.net/Articles/914878/>) (10,000 by default) on the number of times the system can oops or warn before it just panics and reboots. 
  * The `TIOCSTI` `ioctl()` operation will push data into a terminal device; that data will then be read as if were input typed by the user. As one might imagine, attackers find this operation useful. It seems that almost nobody else does, though. In 6.2, the kernel has [gained a configuration option and sysctl knob](<https://git.kernel.org/linus/83efeeeb3d04>) that can disable this functionality entirely. 

#### Internal kernel changes

  * There is a new `struct encoded_page` type meant to encapsulate the idea of using the lower bits of a pointer value for related information. This type, was created by Torvalds to increase type safety and prevent the accidental dereferencing of an augmented pointer without stripping out the extra bits first. There is no documentation but [this commit](<https://git.kernel.org/linus/70fb4fdff582>) is easy enough to read. 
  * The venerable `container_of()` macro has a new sibling called `container_of_const()` that preserves the `const` quality of the passed-in pointer. In [the merge message](<https://git.kernel.org/linus/71a7507afbc3>), Greg Kroah-Hartman explains this macro this way: 

> The driver for all of this have been discussions with the Rust kernel developers as to how to properly mark driver core, and kobject, objects as being "non-mutable". The changes to the kobject and driver core in this pull request are the result of that, as there are lots of paths where kobjects and device pointers are not modified at all, so marking them as "const" allows the compiler to enforce this. 
> 
> So, a nice side affect of the Rust development effort has been already to clean up the driver core code to be more obvious about object rules. 

  * The minimum version of binutils needed to build the kernel has been raised to 2.25. 

One thing that didn't make it this time around is [support for linear address masking](<https://lwn.net/Articles/902094/>), an Intel feature that allows storing extra data in pointer values. Torvalds [complained](<https://lwn.net/ml/linux-kernel/CAHk-=wi=TY3Kte5Z1_nvfcsEh+rcz86pYnzeASw=pbG9QtpJEQ@mail.gmail.com/>) about how the feature was implemented and refused to pull the patches. So this feature, it seems, will have to wait at least another cycle before landing in the mainline. 

Meanwhile, the "extensive changelog" award must certainly go to Christian Brauner, for [this patch](<https://git.kernel.org/linus/11933cf1d91d>), which features 520 lines of explanation (not including the stack trace) for a one-line fix. 

Normally, the 6.2 development cycle would be expected to come to a close on February 12 or 19\. Torvalds suggested that the holidays might slow down this release cycle slightly; time will tell. Meanwhile, it is time to start finding and fixing bugs — once the kernel developers finish celebrating the holidays, of course.
