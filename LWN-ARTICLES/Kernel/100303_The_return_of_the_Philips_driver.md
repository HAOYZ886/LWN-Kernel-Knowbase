---
title: "The return of the Philips driver?"
url: https://lwn.net/Articles/100303/
date: "September 1, 2004"
category: Philips driver
author: 
---

When the Philips webcam driver maintainer requested that driver's removal, the kernel developers complied. The fact remains, however, that the code for the core driver was released under the GPL; it remains out there for those who wish to make use of it. The proprietary "pwcx" decompression code is another story; it has been withdrawn and is unlikely to return. But the GPL code could, perhaps, come back. 

The original maintainer questions the value of the GPL-only code. Without the decompression module, the camera can only be used in a very low-resolution mode. There are a couple of reasons for wanting that code back, however. One of the more interesting ones was [posted](<https://lwn.net/Articles/100306/>) by a member of the [LavaRnd project](<http://www.lavarnd.org/>). It seems that a Philips webcam, with the lens cap in place, is a good source of entropy for random number generators. In fact, the low-resolution stream is even better than the full-resolution version for this application. The LavaRnd folks would like to see the GPL driver back - and they have even volunteered to maintain it. 

The other use for the GPL driver would be as a starting point while the compression protocol is reverse engineered and a completely free driver is created. There has been some speculation that this reverse engineering would be relatively easy - but it will remain speculation until somebody produces some code. 

In any case, the PWC driver is likely to come back in some form; USB maintainer Greg Kroah-Hartman has [stated](<https://lwn.net/Articles/100313/>) that a conversation is in progress with Nemosoft (the original author) and that a patch is forthcoming. Getting a driver which only supports the low-resolution mode is unlikely to please many PWC owners, but it is a start. If the end result of all this is, eventually, a 100% free driver supporting full functionality, everybody will be better off.
