---
title: e1000e and the joy of development kernels
url: https://lwn.net/Articles/300202/
date: "September 24, 2008"
category: "Device drivers-Network drivers"
author: "By Jonathan Corbet September 24, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 24, 2008

The [2.6.27-rc regression list](<http://lwn.net/Articles/299593/>) posted on September 21 contains - deep within the list - an entry reading "e1000e: 2.6.27-rc1 corrupts EEPROM/NVM". One might be forgiven for missing it; the list of regressions is still (unfortunately) long, and there is nothing there to indicate that it is a notable problem. But it is: this particular bug goes beyond breaking networking; when it bites, it corrupts the EEPROM on the device, causing it to cease to function forevermore (or, at least, until the user can manage to flash the EEPROM with working code). This is a problem which is worth fixing. 

As of this writing, though, nobody seems to know what the problem is. There was some confusion resulting from the fact that the related e1000 driver also suffered from an EEPROM corruption problem - but that turns out to have been an entirely different bug. The e1000 problem was fixed by putting a lock around accesses to the EEPROM, preventing corruption caused by concurrent access. But something else is going on with the e1000e. 

Figuring out what that "something else" is appears to be a challenge. The problem is not readily reproducible, and there is this little problem that triggering the bug more than once requires the replacement of the affected hardware. It's not even clear which kernel versions are affected, though it appears that only the 2.6.27 development series shows the bug. There is some correlation between e1000e corruptions and graphics driver crashes, leading David Miller to pursue a [hypothesis](<https://lwn.net/Articles/300208/>) that the real culprit is changes to the X server, but that idea has not, yet been proven. Other developers suspect a concurrency-related problem similar to the e1000 bug. 

As of this writing, the bulk of what is known can be found in [this advisory from Mandriva](<http://blog.mandriva.com/2008/09/23/urgent-notification-major-bug-in-all-mandriva-linux-2009-pre-releases/>). Kernel developers are adding information to [the kernel bugzilla entry](<http://bugzilla.kernel.org/show_bug.cgi?id=11382>) as they find it. 

It has been suggested that anybody running 2.6.27 on a potentially affected system might want to save a copy of the current EEPROM contents with a command like: 

```
ethtool -e eth0 > eth0.eeprom
```

(That assumes, of course, that the relevant device is `eth0` on your system). With the saved data, it should be possible to recover the device if the worst happens; without, chances are that victims will have to return their systems to the vendor. 

In one sense, this bug demonstrates that the system works. It was caught while the kernel was still in the stabilization phase; one can be certain that it will be obliterated somehow before any stable 2.6.27 release comes out. On the other hand, the [first report](<https://lwn.net/Articles/300222/>) of this problem hit the net on August 8; the problem was known for over a month before distributors started responding to it and the all-out hunt for the cause began. That is a long time for any regression to persist, but it is especially long when one is dealing with a regression which has the ability to regress hardware back to a stone-age state. 

The distributors have now responded; most of them have withdrawn kernels with the affected drivers. So far, nobody has posted tools to help affected users recover their hardware (suggestions to use `ibautil` should be ignored and forgotten about as soon as possible). Such a tool [is forthcoming](<https://lwn.net/Articles/300226/>), but it would be hard to blame the relevant engineers for focusing on fixing the problem first. With any luck at all, the root cause will have been isolated by the time you read this. 

There is one thing that will not have changed, though. Testers of unstable software - especially the kernel - have often been warned that said software can do all kinds of terrible things to their systems. It is easy to ignore those warnings; even -rc1 kernels actually work for most people, most of the time. But, as we have seen in this case, the potential for catastrophic bugs is real. Development code can brick your network adapter, scramble your filesystems, open up severe security holes, or save your documents as OOXML. When experimenting with unstable code - even if it has been neatly packaged by your distributor - it is always prudent to have good backups and an even better sense of humor.
