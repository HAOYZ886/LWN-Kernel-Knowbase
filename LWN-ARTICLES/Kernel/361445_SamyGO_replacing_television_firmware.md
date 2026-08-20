---
title: "SamyGO: replacing television firmware"
url: https://lwn.net/Articles/361445/
date: "November 14, 2009"
category: Embedded systems
author: "By Jake Edge November 14, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 14, 2009

While it is quite common for consumer electronics--TVs, DVRs, and the like--to be running Linux these days, it is less common to see projects geared towards replacing and upgrading the Linux firmware in that class of devices. But that is exactly what the [SamyGO](<http://samygo.sourceforge.net/>) project is doing for Samsung televisions. By using the source provided by Samsung, along with quite a bit of ingenuity, SamyGO allows users to telnet into their television--an amusing concept--but also to enable functionality beyond that which ships with the device. 

The SamyGO wiki [lists](<http://sourceforge.net/apps/mediawiki/samygo/index.php?title=Main_Page>) several modifications that can be made to the TV firmware. One of the main modifications seems to be enabling NFS or SMB/CIFS support so that media files from servers on the network can be played. The TVs already support getting media from the local network using [Digital Living Network Alliance](<http://en.wikipedia.org/wiki/Digital_Living_Network_Alliance>) (DLNA) protocols, but there are restrictions on the audio and video formats and some playback functionality (pause, forward, rewind) depending on the DLNA server. By using NFS or CIFS, all of the [formats](<http://sourceforge.net/apps/mediawiki/samygo/index.php?title=Media_Play_and_DLNA>) and features available for USB-based playback are also available across the network. 

Obviously, these are fairly high-end TVs, with both Ethernet connectivity and USB ports. The [devices](<http://sourceforge.net/apps/mediawiki/samygo/index.php?title=Service_Manuals>) "supported" by SamyGO are LCD models in the LE-32-55Bxxx series and LED models from the UE-xx-B70xx series. The USB ports are available for viewing/playing additional media or for games. Using the "Games" menu with programs stored on a USB stick is one of the ways to run programs on the TV. 

The USB ports are also used for a Samsung-branded WiFi "dongle" that owners can buy to avoid the wiring hassle of Ethernet. But, Linux supports far more wireless devices than just the Samsung devices, so SamyGO developers are working to enable others as well. In fact, the Ralink `rt73` and `rt2870` drivers have been [modified](<http://sourceforge.net/apps/phpbb/samygo/viewtopic.php?f=3&t=14#p42>) in the kernel source supplied by Samsung to remove many additional device IDs, so that only the Samsung devices will work. There are now [drivers](<http://sourceforge.net/apps/mediawiki/samygo/index.php?title=How_to_enable_Telnet/NFS/CIFS/SAMBA#WiFi>) available without that restriction. 

The early efforts have been to get telnet working so that the TV filesystem could be explored. This is done by [patching the firmware binaries](<http://sourceforge.net/apps/mediawiki/samygo/index.php?title=How_to_enable_Telnet_on_samsung_TV's>) provided by Samsung and then using the TV's firmware upgrade mechanism to install them on the device. The aptly named "[Warning : Read Me First or Brick Your TV!](<http://sourceforge.net/apps/phpbb/samygo/viewtopic.php?f=1&t=3>)" message in the SamyGO [forum](<http://sourceforge.net/apps/phpbb/samygo/index.php>) outlines the dangers of upgrading the firmware. For those that just want to try this all out, without upgrading any firmware, a safer method is also described, which masquerades as a game on a USB stick to enable telnet. 

The kernel is 2.6.18-based with the addition of Samsung's [Robust FAT File System](<http://www.samsung.com/global/business/semiconductor/products/flash/Products_RFS_Brochure.html>) (RFS), which is a filesystem for NAND flash devices. As the name would indicate, it is also FAT compatible. It is not in the mainline, however, nor have the SamyGO developers gotten it working for desktop distributions. For that reason, they have resorted to binary patching of the firmware. 

Samsung has also released RFS source, along with a [Linux porting guide](<http://www.samsung.com/global/business/semiconductor/products/flash/Products_RFS_PortingGuide.html>) that should be helpful in those efforts. Once RFS can be built for recent kernels, or a utility to create RFS images is made, developers will be able to build their own firmware images for these TVs. [ **Update** : see the comments below, there is no source RFS release. ] 

The kernel source is [available](<http://www.samsung.com/global/opensource/>), but the project has not yet released any kernels built from it. The Ralink drivers were rebuilt after modifying the device IDs, though, so they can be inserted into the system. The kernel itself has been patched, adding OMAP architecture and sound support among other things, but there has been no mention of binary drivers on the forum, so it should be possible to build the released kernel--or something more recent. 

So far, Samsung doesn't seem to have reacted to the project, either positively or negatively. Some concern has been expressed in the forum that working around the WiFi restrictions might raise the company's ire. But one would guess that the number of folks willing to risk bricking an expensive TV in order to use a cheaper WiFi dongle is relatively small--likely to go unnoticed by Samsung. 

In the meantime, if the SamyGO hackers add other functionality that might be interesting to customers--there has been talk of web browsers for example--Samsung might just adopt it themselves. Either way, the code is out there for those who might want to give it a try.
