---
title: The Thunderclap vulnerabilities
url: https://lwn.net/Articles/782381/
date: "March 6, 2019"
category: "Security-Vulnerabilities"
author: "By Jake Edge March 6, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
March 6, 2019

It should come as no surprise that plugging untrusted devices into a computer system can lead to a wide variety of bad outcomes—though often enough it works just fine. We have reported on a number of these kinds of vulnerabilities (e.g. [BadUSB in 2014](<https://lwn.net/Articles/608503/>)) along the way. So it will not shock readers to find out that another vulnerability of this type has been discovered, though it may not sit well that, even after years of vulnerable plug-in buses, there are still no solid protections against these rogue devices. This most-recent entrant into this space targets the [Thunderbolt](<https://en.wikipedia.org/wiki/Thunderbolt_\(interface\)>) interface; the vulnerabilities found have been dubbed "Thunderclap". 

There are several different versions of Thunderbolt, either using Mini DisplayPort connectors (Thunderbolt 1 and 2) or USB Type-C (Thunderbolt 3). According to the long list of researchers behind Thunderclap, all of those are vulnerable to the problems they found. Beyond that, [PCI Express](<https://en.wikipedia.org/wiki/PCI_Express>) (PCIe) peripherals are also able to exploit the Thunderclap vulnerabilities, though they are a bit less prone to hotplugging. Thunderclap is the subject of a [paper [PDF]](<https://thunderclap.io/thunderclap-paper-ndss2019.pdf>) and [web site](<https://thunderclap.io/>). It is more than just a bunch of vulnerabilities, however, as there is a [hardware and software research platform](<https://github.com/thunderclap-io>) that they have developed and released. A high-level [summary of the Thunderclap paper](<https://www.lightbluetouchpaper.org/2019/02/26/struck-by-a-thunderbolt/>) was posted to the Light Blue Touchpaper blog by Theo Markettos, one of the researchers, at the end of February. 

At its core, Thunderclap exploits the ability of devices with direct memory access (DMA) capability to read system memory, including memory that is not at all related to the supposed function of the device. That memory could easily have sensitive information, such as encryption keys or login credentials, that the user probably does not realize they are exposing. In addition, because USB Type-C connectors are used for charging these days, it will be highly surprising that a "charger" may actually also be a Thunderbolt device—with all of the access that implies. Many users may not realize their "innocuous" charging port is really much more than that. 

The primary hardware defense against rogue DMA devices is the [I/O memory-management unit](<https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit>) (IOMMU), but many operating systems do not even enable it. Windows, Linux, and FreeBSD do not enable the IOMMU for Thunderbolt, though Linux has added [some support](<https://lwn.net/ml/linux-kernel/20181112160628.86620-1-mika.westerberg%40linux.intel.com/>) in 5.0. But even for macOS, which does enable the IOMMU, there are significant holes. The researchers created a fake Thunderbolt network card that had far more access than expected on the systems that they tested. Markettos described it this way (some of which found its way into last week's [Security quotes of the week](<https://lwn.net/Articles/780816/>)): 

We found the attack surface available to a network card was much richer and more nuanced than was previously thought. By examining the memory it was given access to while sending and receiving packets, our device was able to read traffic from networks that it wasn't supposed to. This included VPN plaintext and traffic from Unix domain sockets that should never leave the machine. 

On MacOS and FreeBSD, our network card was able to start arbitrary programs as the system administrator, and on Linux it had access to sensitive kernel data structures. Additionally, on MacOS devices are not protected from one another, so a network card is allowed to read the display contents and keystrokes from a USB keyboard. 

Worst of all, on Linux we could completely bypass the enabled IOMMU, simply by setting a few option fields in the messages that our malicious network card sent. 

In theory, the IOMMU could provide the full protection needed, but the current implementations in today's operating systems are lacking: 

Existing strategies for using the IOMMU to protect against DMA attacks come with a high performance cost. This cost has led current operating systems to trade off security for performance gains - in some cases even disabling the IOMMU by default. Current operating systems also put sensitive data in the same regions of memory used for peripheral device communication, which facilitates attacks even when the IOMMU is enabled. 

Markettos likens the situation with the IOMMU to that of the system-call interface. The latter has undergone lots of testing and hardening over the years, but that process is just starting for the IOMMU: 

More generally, since this is a new space of many vulnerabilities, rather than a specific example, we believe all operating systems are vulnerable to similar attacks, and that more substantial design changes will be needed to remedy these problems. We noticed similarities between the vulnerability surface available to malicious peripherals in the face of IOMMU protections and that of the kernel system call interface, long a source of operating system vulnerabilities. The kernel system call interface has been subjected to much scrutiny, security analysis, and code hardening over the years, which must now be applied to the interface between peripherals and the IOMMU. 

Linux does have [support](<https://christian.kellner.me/2017/12/14/introducing-bolt-thunderbolt-3-security-levels-for-gnulinux/>) for Thunderbolt 3 security levels (or access controls), though those are fairly coarse controls. The security levels affect whether certain types of devices can be used with the system, but do not restrict the access of authorized devices. There is [a user-space daemon and a command-line program](<https://gitlab.freedesktop.org/bolt/bolt>) to work with the security levels, as well as [GNOME integration](<https://christian.kellner.me/2019/02/27/thunderclap-and-linux/>). These security levels are also governed by the identification string provided by the device itself, so spoofing is certainly a possibility. 

Protections against the Thunderclap vulnerabilities range from the drastic to more prosaic mechanisms: 

More generally, however, we have discovered a larger vulnerability space that is not fully addressed by mitigations for specific attacks. The best way to fully protect yourself is to disable the Thunderbolt ports on your machine. However, the need to charge and connect legitimate peripheral devices makes doing so infeasible in general. PCs often allow disabling Thunderbolt in firmware (BIOS/UEFI) settings, while permitting power, video and USB. Externally, a USB hub/docking station without Thunderbolt support should suffice to prevent a USB-C port being switched into Thunderbolt mode. 

You can also protect yourself by not leaving your computer unattended in public and not using public USB-C charging stations. Be wary of connecting an unknown device to the Thunderbolt port of your machine, even chargers and projectors that may seem harmless. 

The advice about unattended computers is true for regular USB ports—and [in general](<https://lwn.net/Articles/359145/>)—as well. With each new hardware feature, it seems, the physical security of our computers and other devices becomes ever more important. In some sense, Thunderclap isn't really new, but is simply yet another reminder of the perils of hardware (in)security. While our software has its share of problems, of course, the hardware folks aren't making the security story any easier.
