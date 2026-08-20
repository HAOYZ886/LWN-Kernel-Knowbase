---
title: Attack of the killer CD
url: https://lwn.net/Articles/128741/
date: "March 22, 2005"
category: Security
author: 
---

This story starts to get a little tiresome: a security researcher has found [yet another set of vulnerabilities](<https://lwn.net/Articles/128365/>) in the Linux kernel. The researcher this time is Michal Zalewski, who, in the past, has had great luck finding problems by feeding random data to code. It didn't take him too long to find a few ways to crash the kernel with corrupted CD images. 

The impact of this bug is that anybody who can cause a CD to be mounted can crash the system, and, potentially, obtain root access. Mounting a disk is normally a privileged operation, but many systems are set up to automatically mount a CD (and, perhaps, fire off a file manager window) on insertion. Others are set up to allow unprivileged users to mount a CD on demand. So corrupt CDs are, indeed, a mechanism which could be used to compromise a system. 

Of course, it is true that anybody who gets into a position where they can insert a CD into the system may well find a way to compromise it anyway. It is hard to defend against an attacker with physical access. Even so, there is no point in making any sort of attack easier. 

The bugs in this case are ancient; much of the ISO9660 code dates back to the early 1990's, and it hasn't seen a great deal of maintenance since. In some places, values obtained from the filesystem are not properly checked, leading to inappropriate memory accesses. In one other, the check was in place, but the code responds to a corrupt disk by calling `panic()`, thus creating a nice denial of service situation. There's guaranteed to be other problems which have not yet been found; as Linus [put it](<https://lwn.net/Articles/128744/>), ""The code is a mess."" 

Other filesystems may have similar problems. An on-disk filesystem is a complicated data structure, and it can be very hard to defend against any sort of corruption. Users are plugging in filesystems more frequently; many consumer gadgets, such as cameras and music players, just look like another disk to the computer. So the opportunities for filesystem-based attacks are growing. Expect more patches as more ten-year-old bugs are found and fixed.
