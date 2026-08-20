---
title: The rest of the 7.2 merge window
url: https://lwn.net/Articles/1078539/
date: "June 29, 2026"
category: "Releases-7.2"
author: "By Jonathan Corbet June 29, 2026"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 29, 2026

Linus Torvalds released [7.2-rc1](<https://lwn.net/ml/all/CAHk-=wiMaLpPbO+46RqC1=tYBYt9Z2yPJvswjKBJXh3FDxaaog@mail.gmail.com/>) and closed the 7.2 merge window on June 28; by that time, 13,412 non-merge commits had found their way into the mainline. That makes this the busiest merge window since the 6.7 development cycle in 2024 (15,418 commits, including 2,800 for the entire bcachefs development history). Just under half of those commits arrived after [LWN's summary of the first half of the merge window](<https://lwn.net/Articles/1078068/>) was written. As usual, the commits in the latter part of the merge window were more heavily focused on fixes, but there were still a lot of new features and significant changes merged as well. 

Some of the most interesting changes merged include: 

#### Core kernel

  * The ongoing work to improve the kernel's swap subsystem (previously discussed in [this article](<https://lwn.net/Articles/1056405/>) and [this article](<https://lwn.net/Articles/1057102/>)) continues with the merging of [this series](<https://lwn.net/ml/all/20260517-swap-table-p4-v5-0-88ae43e064c7@tencent.com>) unifying much of the code dealing with anonymous and shared-memory folios and reducing the memory usage of the swap subsystem itself. ""The static metadata overhead is now close to zero, and workload performance is slightly improved"". See [our LSFMM+BPF 2026 coverage](<https://lwn.net/Articles/1072657/>) for more information on this work. 
  * The `khugepaged` kernel thread can now create multi-size transparent huge pages (mTHPs) automatically. See [this article](<https://lwn.net/Articles/1077208/>) for details on this work. 
  * A separate option to create transparent huge pages in the page cache for read-only files — a workaround for filesystems that do not support large folios natively — [has been removed](<https://lwn.net/Articles/1066582/>). This change may slow performance for some filesystems until they gain large-folio support. 

#### Filesystems and block I/O

  * The default block size for NFS transfers has increased to 4MB on systems with at least 16GB of RAM. 
  * The SMB filesystem server has gained support for files that are stored in compressed format; it also now implements compression for data sent (or received) over the network. 
  * The [newly reinvigorated NTFS filesystem](<https://lwn.net/Articles/1055062/#ntfs>) has been hardened against many types of on-disk metadata corruption and is now able to handle Windows-native symbolic links. 
  * The fscache backend for the EROFS filesystem, which was [merged for 5.19](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=65965d9530b0>), has been deprecated for the last two years; it has now been removed. 
  * The Ceph filesystem has gained the ability to reset clients manually. This feature is rigorously undocumented, but [the patch series](<https://lwn.net/ml/all/20260507122737.2804094-1-amarkuze@redhat.com/>) contained an informative cover letter that was not preserved when the series was applied. 

#### Hardware support

  * **Clock** : Canaan Kendryte K230 clocks, Qualcomm IPQ9650 global clock controllers, Qualcomm Hawi TCSR clock controllers, Qualcomm X1P42100 video clock controllers, and Qualcomm X1P42100 camera clock controllers. 
  * **GPIO and pin control** : Qualcomm IPQ9650 pin controllers, Qualcomm Nord pin controllers, Qualcomm SM6350 LPASS LPI pin controllers, Qualcomm Shikra pin controllers, Aspeed G7 SoC pin controllers, NVIDIA Tegra238 and Tegra264 pin controllers, and UltraRISC DP1000 SoC pin controllers. 
  * **Industrial I/O** : Analog Devices AD5706R digital-to-analog converters, MEMSIC MMC5983 3-axis magnetic sensors, Broadcom APDS9999 ALS, RGB, and proximity sensors, Analog Devices AD4691 analog-to-digital converters, and Vishay VEML3328 RGBCIR light sensors. 
  * **Input** : Rakk gaming peripherals, OneXPlayer handheld controllers, and Wacom W9000-series pen-abled touchscreens. 
  * **Media** : AMD ISP4 image signal processors, AVMatrix HWS PCIe image-capture devices, and Intel CVS CSI-2 bridges. 
  * **Miscellaneous** : Renesas R-Car multi-functional interfaces, NVIDIA Tegra114 external memory controllers, Verisilicon I/O memory-management units, TI LP5860 LED controllers, Samsung S2M series RGB and flash/torch LEDs, Maxim MAX25014 controllers, Microsoft Surface RT batteries, Samsung S2M series PMIC battery chargers, AMD Promontory 21 xHCI temperature sensors, Qualcomm SHIKRA, HAWI, and Nord interconnect providers, Qualcomm Hamoa/Glymur reference-device embedded controllers, Dell DW5826e PLDR reset controllers, and UltraRISC PCIe host controllers. 
  * **Networking** : Virtio CAN interfaces. 
  * **PHY** : Axiado eMMC PHYs, Mobileye EyeQ5 Ethernet PHYs, EcoNet PCIe PHYs, TI DS125DF111 2-Channel retimers, Freescale Layerscape Lynx 10G SerDes PHYs, and NXP TJA1145 CAN transceiver PHYs. 
  * **USB** : Cadence USBSSP USB device controllers and devices supporting the USB4STREAM protocol. 

#### Miscellaneous

  * The `perf` tool has, as usual, received a long list of improvements; see [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=05d2a3da153b>) for details. 

#### Security-related

  * The integrity measurement architecture (IMA) has gained the ability to stage measurement data outside of the kernel, thus freeing (possibly large amounts of) kernel memory. See [Documentation/security/IMA-export-delete.rst](<https://docs.kernel.org/next/security/IMA-export-delete.html>) for more information. 
  * The [Landlock](<https://docs.kernel.org/next/security/landlock.html>) security module now has the ability to control the use of UDP sockets; [this documentation commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=299eccf99668>) shows what has changed. It is also now possible to suppress logging for some Landlock denials, [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=29752205db5f>) contains some more information. 
  * After six years of effort, the `strncpy()` implementation has been removed from the kernel; according to [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1a3746ccbb0a>), this effort required 362 commits from 70 contributors. [This commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=079a028d6327>) lists the alternatives in use by the kernel now. 
  * One notable thing that was _not_ merged is the Hornet Linux security module, which was intended to provide an additional level of security around the loading of BPF programs. This module was [rejected](<https://lwn.net/Articles/1042625/>) by the BPF developers in 2025, but was [reworked](<https://lwn.net/ml/all/20260507191416.2984054-1-bboscaccy@linux.microsoft.com/>) in an attempt to satisfy the objections that had kept it out previously. That effort was not successful, though, and, prior to the beginning of the merge window, Torvalds [let it be known](<https://lwn.net/ml/all/CAHk-=wj6CtZS9hbwFjQcoNkPwQLoyKmk8czaBF6=bBOCYuXEUQ@mail.gmail.com>) that he would not be accepting a pull request containing that code. 

Where things go from here is not clear; Blaise Boscaccy and other developers in this area clearly feel that Hornet provides security guarantees that are not present with the in-kernel solution. Danial Borkmann has posted [an alternative implementation](<https://lwn.net/ml/all/20260624140301.93421-1-daniel@iogearbox.net>) that, he hoped, might constitute an acceptable compromise. That work is in an early state, though, and is not likely to go upstream in the near future. 

#### Virtualization and containers

  * The KVM subsystem has gained support for Intel's "mode-based execution control" (MBEC) and AMD's "guest-mode execution trap" features, both of which give better control over execute permissions in guests. Among other things, this feature greatly reduces the number of times that guests must trap back into the host for permission-related operations. See [this MBEC patch series](<https://lwn.net/ml/all/20251223054806.1611168-1-jon@nutanix.com/>) (which was substantially reworked before inclusion) and [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2be108307eae>) for more information. 
  * There are no new KVM features for the Arm architecture this time around; from [the KVM pull request](<https://lwn.net/ml/all/20260618152747.132796-1-pbonzini@redhat.com/>): ""This is a bit of an odd merge window on the KVM/arm64 front. There is absolutely no new feature in the pull request. It is purely fixes, because it is simply becoming too hard to review new stuff when so many AI-fueled fixes hit the list"". 

#### Internal kernel changes

  * The kernel build system has gained the ability to create a software bill of materials (SBOM) using the [SPDX](<https://spdx.dev/>) information that has been [added to the kernel's source files](<https://lwn.net/Articles/739183/>). Only the files that are actually compiled under the current configuration are considered, so the SBOM is specific to the built kernel. The relevant command is "`make sbom`"; see [Documentation/tools/sbom/sbom.rst](<https://docs.kernel.org/next/tools/sbom/sbom.html>) for more information. 

The 7.2 merge window removed 195 exported symbols and added 380 others; there were also 32 new kfuncs added. See [this page](<https://lwn.net/Articles/1079908/>) for the full list. 

Finally, the 7.2 merge window included contributions from 2,138 contributors, which is almost certainly a merge-window record. It is only the last three development cycles that have seen that many contributors for the entire cycle. Of the 7.2 merge-window contributors, 404 were first-timers. There were 706 commits (about 5% of the total) containing Assisted-by tags. It would seem that the LLM-fueled flood of kernel patches is showing no signs of slowing down. 

Now all of those changes need to be stabilized for the 7.2 release, which is most likely to happen on August 16.
