---
title: Configuration of pluggable network adaptors
url: https://lwn.net/Articles/103203/
date: "September 22, 2004"
category: Hotplug; Networking
author: 
---

Li Shaohua [ran into a problem](<https://lwn.net/Articles/103204/>) when repeatedly plugging and unplugging an e1000 network adaptor. After 32 times, the adaptor would no longer work. It seems that the driver (like many others in the 2.6 kernel) was designed to discover at most 32 devices at boot time, and it has space for configuration parameters for just that many devices. Each new hotplug event looked like a new device, so the driver quickly ran out of parameter storage. In fact, the e1000 driver can handle many more devices than that; it just lacks space in its boot-time arrays to hold default configuration information. 

Mr. Li's diagnosis was that the problem lies with the e1000 driver's inability to reuse board numbers internally. So he wrote up a patch to keep track of existing boards, and to reuse their numbers when they are removed. After some discussion, this patch was [reworked](<https://lwn.net/Articles/103205/>) into a general mechanism using the "idr" facility (described in the next article) - since the e1000 is not the only driver which behaves this way, it makes sense to fix the problem once for everybody. 

[Not everybody agrees](<https://lwn.net/Articles/103207/>) that this is the right approach, however. Boot-time configuration parameters can be useful for many (if not most) systems where the network interfaces are screwed down and are unlikely to be replaced while the system is up. But do they really make sense for hotpluggable devices? There is a whole system in place for the configuration of hotpluggable devices; perhaps that should be used rather than adding complexity to the network drivers. Given that the conversation came to a hard stop after this view was posted, it seems likely to carry the day.
