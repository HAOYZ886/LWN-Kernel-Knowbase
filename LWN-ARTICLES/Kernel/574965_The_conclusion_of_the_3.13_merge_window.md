---
title: The conclusion of the 3.13 merge window
url: https://lwn.net/Articles/574965/
date: "November 26, 2013"
category: "Releases-3.13"
author: "By Jonathan Corbet November 26, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 26, 2013

Linus [released 3.13-rc1](<https://lwn.net/Articles/574781/>) and closed the 3.13 merge window on November 22, perhaps a couple of days earlier than some developers might have expected. Counting a couple of post-rc1 straggler pulls, some 10,600 non-merge changesets were pulled into the mainline for this development cycle; that is about 700 since [last week's summary](<https://lwn.net/Articles/574222/>). 

As might be expected, the list of user-visible features included in that relatively small set of patches is short; it includes: 

  * The squashfs filesystem now has multi-threaded decompression; it can also decompress directly into the page cache, eliminating the temporary buffer used previously. 

  * There have been several changes to the kernel's key-storage subsystem. The maximum number of keys has increased to an essentially unlimited value, allowing, for example, the NFS code to store vast numbers of user ID mapping values as keys. There is a new concept of a "trusted" key, being one obtained from the hardware or otherwise validated, and keyrings can be marked as allowing only trusted keys. Finally, a mechanism for persistent keys not attached to a given user ID has been added, and key data can be quite large; both of these changes were needed to enable Kerberos to use the key subsystem. 

  * New hardware support includes: 

    * **Input** : Samsung SUR40 touchscreens. 

    * **Security** : Nuvoton NPCT501 I2C trusted platform modules, Atmel AT97SC3204T I2C trusted platform modules, OMAP34xx random number generators, Qualcomm MSM random number generators, and Freescale cryptographic accelerators (job ring support).  

Changes visible to kernel developers include: 

  * There is a new associative array data structure in the kernel. It was added to support the keyring work, but could be applicable in other situations as well. See [Documentation/assoc_array.txt](<https://lwn.net/Articles/574966/>) for details. 

  * The information in `struct page` is now even more dense with the addition of Joonsoo Kim's patch set to have the slab allocator store more information there. See [this article](<https://lwn.net/Articles/565097/>) for details. 

Now the final stabilization phase for all of this work begins. Your editor predicts that the final 3.13 kernel will be released sometime between the New Year and the beginning of [linux.conf.au 2014](<http://linux.conf.au/>) on January 6.
