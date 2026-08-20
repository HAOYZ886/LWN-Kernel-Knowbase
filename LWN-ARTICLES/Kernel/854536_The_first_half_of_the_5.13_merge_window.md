---
title: The first half of the 5.13 merge window
url: https://lwn.net/Articles/854536/
date: "April 30, 2021"
category: "Releases-5.13"
author: "By Jonathan Corbet April 30, 2021"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 30, 2021

As of this writing, just over 7,800 non-merge commits have been pulled into the mainline repository for the 5.13 development cycle. It does indeed seem true that 5.13 will be busier than its predecessor was. The work merged thus far affects subsystems across the kernel; read on for a summary of what has been merged so far. 

#### Architecture-specific

  * Some of the preparatory work to support Apple's M1 processor has been merged. 

#### Core kernel

  * There is a new "misc" control-group controller that is meant to handle simple resources that can be managed with a straightforward counter. Its first use is to manage address-space IDs used with AMD's secure encrypted virtualization (SEV) mechanism. Some information can be found in [this documentation commit](<https://git.kernel.org/linus/25259fc914a1>). 
  * The poll operation for [io_uring](<https://lwn.net/Articles/776703/>) now has a "multishot" mode. Normally, a poll (like any other io_uring operation) is removed from the ring once it generates a completion event. A multishot poll, instead, remains active and, uniquely, can generate multiple completion events from a single submitted event. 

#### Filesystems and block I/O

  * The new [EROFS](<https://www.kernel.org/doc/html/latest/filesystems/erofs.html>) filesystem "big pcluster" feature provides significantly improved compression performance. 
  * The new `rasize=` SMB3 mount option allows the size of the readahead window to be increased, which can improve performance for some workloads. 

#### Hardware support

  * **Graphics** : generic USB-connected displays, Chipone ICN6211 MIPI-DSI/RGB converter bridges, and Lontium LT8912B DSI/HDMI bridges. 
  * **Hardware monitoring** : Infineon IR36021 power-supply controllers, ST STPDDC60 multicell controllers, NZXT Kraken X42/X51/X62/X72 liquid coolers, BluTek BPA-RS600 power supplies, FSP/3Y-Power power supplies, and Maxim MAX15301 regulators. 
  * **Industrial I/O** : Bosch BMI088 accelerometers, Texas Instruments ADS131E04, ADS131E06 and ADS131E08 analog-to-digital converters, and ChromeOS EC MKBP proximity sensors. 
  * **Miscellaneous** : Nuvoton WPCM450 interrupt controllers, Renesas 79RC3243x interrupt controllers, Microsoft Surface detachment systems, Microsoft Surface power supplies, Microsoft Surface batteries, Advantech ACPI software buttons, Gigabyte WMI temperature sensors, Broadcom NVRAM controllers, Synopsys DesignWare xData PCIe controllers, Microchip Sparx5 SerDes Ethernet PHYs, Marvell CP110 UTMI USB PHYs, Qualcomm SDM660 and SM8350 interconnects, Broadcom 8250-based serial ports, Xilinx USB controllers, Apple interrupt controllers, HiSilicon SPI controllers on Kunpeng SoCs, NXP i.MX8 QXP/QM JPEG encoder/decoders, Rohm BD71817, BD9576MUF, and BD9573MUF power-management ICs, Actions Semi ATC260x power-management ICs, Netronix embedded controllers, Rockchip RK3568 clock controllers, and MediaTek MT7621 clock controllers. 
  * Many old drivers have been removed, including those for wimax devices, the Google "gasket" driver framework, the "sysace" and "umem" block drivers, and a number of old serial-port drivers. 
  * It's also worth noting that the "comedi" data-acquisition drivers have graduated out of the staging tree after being resident there for 13 years and having over 8,700 patches applied. 

#### Security-related

  * The kernel's "trusted keys" mechanism has been made more generic to allow for trusted keys to originate from sources other than the trusted platform module, which is not present on all systems. Specifically, code running within a trusted execution environment can also supply trusted keys under the control of a new `trusted.source` command-line parameter. Some more information can be found in [this documentation patch](<https://git.kernel.org/linus/c429805fdf70>). 
  * The kernel is now able to manage trusted keys in the ASN.1 format. 
  * Until now, the kernel would ignore certificates in the UEFI secure boot blacklist if they were in the `EFI_CERT_X509_GUID` format; that could allow the system to boot with a revoked key. This problem, known as CVE-2020-26541, has been fixed for 5.13 (with backports likely to follow). There is also a new kernel configuration option that allows revocation certificates to be preloaded at boot time. 
  * The kernel's cryptographic layer can now perform elliptic curve digital signature algorithm (ECDSA) verification. 
  * The kernel can now randomize the exact location of the kernel stack on every system call, making exploits harder; see [this commit](<https://git.kernel.org/linus/39218ff4c625>) for details. 
  * Support for Clang's [control-flow integrity](<https://lwn.net/Articles/810077/>) mechanism has been merged. From [this commit](<https://git.kernel.org/linus/cf68fffb66d6>): 

> With CONFIG_CFI_CLANG, the compiler injects a runtime check before each indirect function call to ensure the target is a valid function with the correct static type. This restricts possible call targets and makes it more difficult for an attacker to exploit bugs that allow the modification of stored function pointers. 

CFI is only available for the arm64 architecture currently, but x86 support is in the works. 
  * The SELinux security module can use the kernel's integrity measurement architecture (IMA) to verify that its policies have not been tampered with; [this commit](<https://git.kernel.org/linus/2554a48f4437>) contains a bit of information on how to use this feature. 

#### Internal kernel changes

  * The software-interrupt processing code from the realtime preemption tree has made it into the mainline. In realtime mode, all software interrupts are handled in kernel threads, where they can be preempted by a higher-priority process like anything else. 
  * There is a new helper library to handle many of the common functions needed by network filesystems; see [this documentation patch](<https://git.kernel.org/linus/fb28afccdb97>) for more information. 
  * The `logbuf_lock` used deep within `printk()` has been removed, getting the kernel one step closer to a lockless `printk()` implementation. 

A quick check shows that there are currently nearly 6,400 commits in linux-next that have not yet been pulled into the mainline; that suggests that the 5.13 merge window is far from over. By the usual schedule, the merge window can be expected to close on May 9, with the 5.13 release happening at the end of June or beginning of July. As usual, LWN will catch up with the rest of the merge window once it closes, stay tuned.
