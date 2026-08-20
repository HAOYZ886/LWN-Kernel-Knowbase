---
title: ABI stability documentation
url: https://lwn.net/Articles/173740/
date: "February 28, 2006"
category: "Development model-User-space ABI"
author: 
---

Last week's Kernel Page [looked at the stability of the user-space interface](<http://lwn.net/Articles/172986/>), especially regarding areas like sysfs, which are not always regarded as being part of the kernel ABI. This week, Greg Kroah-Hartman has made an attempt to make the issue more evident through [a set of ABI stability documents](<http://lwn.net/Articles/173661/>). Included in his patch is a proposal for a different way of looking at ABI stability issues. 

Linus has, in the recent past, [taken a hard line](<http://lwn.net/Articles/172989/>) on changes interfaces to user-space: 

If you cannot maintain a stable kernel interface, then you damn well should not send your patches in for inclusion in the standard kernel. Keep your own "HAL-unstable" kernel and ask people to test it there. 

It really is that easy. Once a system call or other kernel interface goes into the standard kernel, it stays that way. It doesn't get switched around to break user space. 

Greg, has, instead, taken the approach that not all kernel interfaces should be seen as stable from the outset. So he has proposed five different classifications for ABI stability: 

  * **Stable**. Interfaces classified as stable will not break "for at least two years," and probably quite a bit longer. The Linux system call interface is classified in this way. 

  * **Testing**. A "testing" interface is one which has been through most of the development process. It is not expected to change, but, that notwithstanding, the possibility of an incompatible change before the interface becomes "stable" does exist. This is the time for user-space programs to begin to make real use of the interface, but user-space developers need to pay attention to what is happening on the kernel side. The sysfs files under `/sys/class` have been designated as having a "testing" level of stability by Greg's documentation. 

  * **Unstable**. This classification is for relatively new interfaces which are expected to change as problems in the initial implementation become clear. Sysfs files under `/sys/devices` are classified as "unstable." 

  * **Private**. This class describes interfaces which are intended to be hidden behind a user-space library and which should not be used directly by applications. The ALSA sound system is an example of a "private" interface. 

  * **Obsolete** marks interfaces which are destined to be removed, and which should not be used at all. Few long-timer observers will be surprised to see that Greg marked devfs as being obsolete. 

Linus [doesn't like the unstable and private classifications](<https://lwn.net/Articles/173757/>), calling them "excuses for bad habits." But it is true that inclusion in the mainline can stress an interface in surprising ways, leading to a need for changes. Interface design is hard, even if you don't have to get everything right the first time. So it may make some sense to allow unstable interfaces into the kernel for a short while - as long as they are clearly documented as such. Thus far, there has been no way to warn developers that a certain interface, perhaps, shouldn't be relied upon quite yet. 

The notion of private interfaces looks harder to justify. There has been some talk of shipping user-space libraries for private interfaces with the kernel, just to help ensure that the whole package provides a stable application interface for any release. That seems like a fairly unlikely change, however, at least for big interfaces like ALSA. 

Changes will likely be made (this scheme might be classified "unstable" at this point), but it seems probable that it will, in some form, be adopted. That can only be a good thing for people interested in a stable user-space interface; once the expectations have been reasonably well documented, it will be easier to live up to them.
