---
title: SDIO support coming
url: https://lwn.net/Articles/242744/
date: "July 24, 2007"
category: SDIO
author: 
---

The Secure Digital Input/Output specification enables the creation of SD cards which handle tasks beyond the simple storage of bits, which is what SD has traditionally been used for. The [SD Association SDIO page](<http://www.sdcard.org/sdio/index.html>) shows some cute pictures with SDIO network adapters, cameras, GPS receivers, fingerprint recognizers, and a strangely disturbing image of a scanner glued directly to an SD card. As small gadgets with SD slots become more prevalent, one can imagine a number of uses for peripherals which can be attached to those slots. Since many of those gadgets run Linux, it would be nice to have proper support for SDIO devices in the mainline kernel. Unfortunately, like much of the SD Association's work, SDIO has been a realm of proprietary specifications and implementations. 

That would appear to be about to change, however: Pierre Ossman has sent out [an announcement](<https://lwn.net/Articles/242747/>) of interest: 

I am happy to announce that SDIO support will soon be a standard feature in Linux. No more proprietary stacks with all the troubles (legal and technical) that go with them. 

The new SDIO stack, written by Pierre and Nicolas Pitre, is in a fairly complete state with all the sorts of bus-level support that driver writers have come to expect. There is one driver (for GPS interfaces) available now; it is expected that others will show up shortly. If all goes well, expect the new SDIO stack to be ready for 2.6.24.
