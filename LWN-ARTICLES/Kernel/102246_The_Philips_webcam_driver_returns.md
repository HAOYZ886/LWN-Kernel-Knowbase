---
title: The Philips webcam driver returns
url: https://lwn.net/Articles/102246/
date: "September 15, 2004"
category: Philips driver
author: 
---

The removal of the Philips webcam driver from the kernel set off a long and sometimes inflammatory discussion. Its return has, instead, been greeted with almost total silence. Once people take a look, however, they might see something worth yelling about. 

The new maintainer is Luc Saillard. He has posted [a patch](<https://lwn.net/Articles/101998/>) which restores the PWC driver to the kernel, but without the problematic hook for the proprietary compression module. As an added bonus, the driver can deal with compressed streams from some cameras (those using chipsets 2 or 3), in some modes. Work still needs to be done for chipset 1 and the Bayer mode. 

The final result is yet to be seen, but it would appear that the whole PWC episode is heading toward a best-case conclusion: a 100% free driver. It would be hard to see that outcome as anything but a good thing.
