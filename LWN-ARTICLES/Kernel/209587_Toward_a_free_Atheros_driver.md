---
title: Toward a free Atheros driver
url: https://lwn.net/Articles/209587/
date: "November 15, 2006"
category: "Devicescape stack; Networking-Wireless"
author: 
---

The Atheros family of wireless chipsets finds its way into a number of network adapters and laptop systems. It is a flexible and capable device, with one little limitation: there is no free Linux driver available. Linux support can be had via the freely-downloadable [MadWifi driver](<http://madwifi.org/>), but, at the core of that driver, there is a binary-only "hardware access layer" (HAL) module which does much of the real work. This module has all of the problems associated with proprietary drivers: it cannot be audited or fixed, it cannot be improved, it is only available for the kernel versions and architectures supported by the manufacturer, etc. But, for Linux users, the choices are MadWifi or nothing. 

A free Atheros HAL module called "ar5k," written by Reyk Floeter, has been in circulation for a couple of years; OpenBSD uses it. But this code has long been followed by allegations that it was improperly developed and potentially subject to copyright claims by Atheros. In the current climate, nobody wants to risk bringing possibly tainted code into the kernel; the potential consequences are just too severe. So, while the desire to support Atheros devices in Linux remains strong, the existing HAL has not been considered and little work has been done to bring that about. 

Except that, as it turns out, work has been quietly happening in an unexpected place. The Software Freedom Law Center was asked by the ar5k developers to look at the development history of the code and come up with a pronouncement on whether it was legitimate (from a copyright law perspective) or not. On November 14, the SFLC produced [its answer](<http://lwn.net/Articles/209472/>): 

SFLC has made independent inquiries with the OpenBSD team regarding the development history of ar5k source. The responses received provide a reasonable basis for SFLC to believe that the OpenBSD developers who worked on ar5k did not misappropriate code, and that the ar5k implementation is OpenBSD's original copyrighted work. 

This finding should clear the way for the entry of the free Atheros HAL into the Linux kernel - eventually. But there are a couple of problems which need to be overcome first. 

One of those is the general level of upheaval in the Linux wireless subsystem. The developers still intend to move over to the Devicescape stack and to get that code into the mainline, but there is still work to be done in that area. But a new wireless driver which does not work with Devicescape will have a harder path into the kernel. There is an effort to move MadWifi over to Devicescape (it's called "DadWifi"), so that might be the quickest path for Atheros support to get into the kernel. 

The other problem, however, is that code based on the HAL concept tends to be unpopular at best. A HAL is typically seen as an unnecessary abstraction layer between the driver and the hardware which serves to obscure what's really going on while adding no real value of its own. So developers who propose HAL-based drivers are usually told to go away and come back once the HAL is gone. There is no real reason to expect things to happen differently this time around. 

But, even if it can't be used directly, the ar5k code is now fair game for reference and eventual adaptation into a Linux driver. There are enough developers out there with an interest in making Atheros adapters work that the chances of this work getting done in the (relatively) near future are relatively good. The list of devices which are not supported by Linux is about to get shorter.
