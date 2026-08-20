---
title: I/O space write barriers
url: https://lwn.net/Articles/103183/
date: "September 22, 2004"
category: IO memory; mmiowb
author: 
---

Some platforms, it seems, have an interesting property: writes to I/O memory space from multiple processors may be reordered before reaching the device. Even if the device registers are protected by a lock (pretty much necessary to keep multiple processors from writing simultaneously and confusing the device), writes issued by one CPU can arrive before those from another, even if the second CPU had held the lock and issued its writes first. The Itanium architecture in particular behaves this way, though others may as well. 

[The answer, according to Jesse Barnes](<https://lwn.net/Articles/103159/>) is the addition of a new type of memory barrier to force the ordering of writes to the device. Jesse's patch adds a new function, `mmiowb()`, which implements this barrier. He has also updated the qla1280 driver to make use of it. 

Authors of PCI drivers are accustomed to coding a different sort of barrier: reading from a device register to ensure that all writes have actually been posted to the device. `mmiowb()` is a different, lighter-weight mechanism. After a call to `mmiowb()`, writes might _still_ have not reached the device. Writes are not forced out; they just have their ordering with respect to subsequent writes guaranteed. In many situations, that sort of guarantee is all that is needed.
