---
title: The first half of the 4.19 merge window
url: https://lwn.net/Articles/762566/
date: "August 17, 2018"
category: "Releases-4.19"
author: "By Jonathan Corbet August 17, 2018"
---

By **Jonathan Corbet**  
August 17, 2018

As of this writing, Linus Torvalds has pulled just over 7,600 non-merge changesets into the mainline repository for the 4.19 development cycle. 4.19 thus seems to be off to a faster-than-usual start, perhaps because the one-week delay in the opening of the merge window gave subsystem maintainers a bit more time to get ready. There is, as usual, a lot of interesting new code finding its way into the kernel, along with the usual stream of fixes and cleanups. 

#### Core kernel

  * The scheduler's load-tracking subsystem has been enhanced with an improved awareness of the amount of time taken by realtime processes, deadline processes, and interrupt handling; this information is used to select more appropriate operating frequencies for the system's processors. 
  * The "jprobes" tracing mechanism has been removed from the kernel; it has long been superseded by the ftrace infrastructure. Those who are curious about what jprobes did can find a description in [this 2005 article](<https://lwn.net/Articles/132196/>). 
  * The [asynchronous I/O polling interface](<https://lwn.net/Articles/743714/>) has been added again, after having been reverted out of 4.18. The internal implementation has changed into a more Linus-friendly form, so this feature should actually make it into the release this time around. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Architecture-specific

  * Support for Intel's "cache pseudo locking" feature has been added. With this feature, a portion of a processor's memory cache can be populated with data of interest, then locked against further changes. The result is consistent low-latency read access to the locked memory range. See [this commit](<https://git.kernel.org/linus/e17e733070d4ab312a35848ab248e85b78dcb3f4>) for documentation on this feature. 
  * 32-Bit x86 systems finally have [kernel page-table isolation](<https://lwn.net/Articles/741878/>) support. 
  * A large set of mitigations for the recently disclosed [L1TF vulnerability](<https://lwn.net/Articles/762570/>) has been merged. 
  * The arm64 architecture has gained support for restartable sequences and the "stackleak" GCC plugin. 

#### Filesystems and block layer

  * The XFS filesystem has removed the `barrier` and `nobarrier` mount options. Those options have not actually done anything for years; hopefully everybody has removed them from their `fstab` files by now. 
  * The [block I/O latency controller](<https://lwn.net/Articles/758963/>) has been added; it allows administrators to provide minimum I/O latency guarantees to specific control groups. 
  * The asynchronous bsg (SCSI generic) interface [has been removed](<https://lwn.net/Articles/760345/>) due to persistent and unfixable design issues. 

#### Hardware support

  * **Audio** : Realtek RT5682 codecs, Everest ex7241 codecs, Amlogic AXG sound cards, and Qualcomm WCD9335 codecs. 
  * **Clock** : Renesas R9A06G032 clock controllers, Maxim 9485 programmable clock generators, Meson AXG audio clock controllers, Actions Semi S700 SoC clock controllers, and Qualcomm SDM845 display clock controllers. 
  * **Graphics** : Ilitek ILI9881C-based panels, Iletek ILI9341 display panels, and Qualcomm SDM845 display processing units. 
  * **Hardware monitoring** : Mellanox fan controllers, Maxim MAX34451 voltage/current monitors, and Nuvoton NPCM750 PWM and fan controllers. 
  * **Media** : Dongwoon DW9807 lens voice coils, Asahi Kasei Microdevices AK7375 lens voice coils, and Socionext MN88443x demodulators. 
  * **Network** : Vitesse VSC7385/7388/7395/7398 switches, Realtek SMI Ethernet switches, and Theobroma Systems UCAN interfaces. 
  * **Pin control** : Intel Ice Lake pin controllers, NXP IMX8MQ pin controllers, and Synaptics as370 pin controllers. 
  * **Miscellaneous** : NVIDIA Tegra NAND flash controllers, Socionext UniPhier SPI controllers, Qualcomm last-level cache controllers, Qualcomm RPMh regulators, Hisilicon SEC crypto block cipher accelerators, Mediatek MT7621 GPIO controllers, and MediaTek CMDQ mailbox controllers. 

#### Networking

  * The [time-based packet transmission](<https://lwn.net/Articles/748879/>) patch set has been merged. This feature allows a program to schedule data for transmission at some future time. 
  * The [CAKE queuing discipline](<https://lwn.net/Articles/758353/>), which works to overcome bufferbloat and other problems associated with home network links, has been merged. 
  * The new "skbprio" queuing discipline can schedule packets according to an internal priority field. This feature is naturally undocumented; in [the commit adding it](<https://git.kernel.org/linus/aea5f654e6b78a0c976f7a25950155932c77a53f>) the author says: ""Skbprio was conceived as a solution for denial-of-service defenses that need to route packets with different priorities as a means to overcome DoS attacks"". 
  * Devices that can offload the receive side processing of TLS-encrypted connections are now supported. 

#### Security-related

  * There is now [a kernel configuration option](<https://lwn.net/Articles/760584/>) that can be used to make the system fully initialize the entropy pool from the hardware random-number generator at boot time. This should allow for better early-boot random-number generation at the cost of placing a bit of trust in the CPU manufacturer's hardware. 

#### Internal kernel changes

  * The [simple wait queue](<https://lwn.net/Articles/577370/>) API has been changed by renaming a number of functions to reflect the fact that it only implements exclusive waits. So `prepare_to_swait()` becomes `prepare_to_swait_exclusive()`, `swake_up()` becomes `swake_up_one()`, and so on. 
  * There is a new initiative to translate kernel documentation into Italian, with [an initial set of translations](<http://static.lwn.net/kerneldoc/translations/it_IT/index.html>) merged for 4.19. 

If the usual schedule holds, the 4.19 merge window can be expected to remain open until August 26\. There are still quite a few trees to be pulled, so one can expect a number of interesting changes will still find their way into this merge window. The final 4.19 release can be expected in mid-October.
