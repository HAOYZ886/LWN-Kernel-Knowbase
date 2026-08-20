---
title: The end of init_etherdev() and friends
url: https://lwn.net/Articles/73897/
date: "March 2, 2004"
category: Device drivers; Networking
author: 
---

The last few 2.6 kernel releases have seen a lot of patches removing calls to a set of network driver support functions, including `init_etherdev()`, `init_netdev()`, and `dev_alloc()`. With the integration of networking and sysfs, static `net_device` structures have become impossible to use in a safe way; these structures must now be allocated dynamicly and properly reference counted. See [this Driver Porting Series article](<http://lwn.net/Articles/30107/>) for details on the currently supported interface. 

As of 2.6.3, there are no users of those functions in the mainline kernel tree. There are, however, certain to be out-of-tree drivers which still use them. Those drivers will need to be fixed soon; the 2.6.3-mm4 kernel tree added a patch which removes those functions forevermore. Once that patch works its way into the mainline kernel, any driver relying upon `init_etherdev()` and friends will cease to work until it is fixed. Don't say you haven't been warned.
