---
title: The death and possible rebirth of sysctl()
url: https://lwn.net/Articles/204935/
date: "October 18, 2006"
category: "Development model-User-space ABI; Sysctl"
author: 
---

The `sysctl()` system call has had a rough life. It began as an idea imported from BSD; it allows a user-space process to tweak various kernel parameters using a set of integer indexes. People quickly discovered, however, that a text and filesystem-based interface (as seen under `/proc/sys`) is much easier to deal with. The `/proc/sys` hierarchy can be adjusted from the shell and manipulated by scripts - and nobody has to worry about sysctl numbers. So there are very few users of `sysctl()`, which has been considered deprecated for a long time. Recent kernels have issued warnings when `sysctl()` is called. 

The 2.6.19-rc kernels take things one step further: for most configurations, `sysctl()` disappears altogether. In a strange sort of turnaround, only configurations with the "embedded" option set can enable `sysctl()` at all. This is all in accordance with the feature removal schedule, which calls for `sysctl()` to go away in January, 2007. 

But `sysctl()` is part of the user-space API, which is never supposed to be broken for any reason. The removal of this function would appear to be a violation of the oft-repeated promise to keep this interface stable. So some developers have started to complain about the API change. There have been calls to back it out again, and to restore `sysctl()` to normal configurations. As Alan Cox [put it](<https://lwn.net/Articles/204943/>): ""We added it, we supported it, we get to keep it. We just stick notes in the docs saying 'please use /proc instead'."" 

Patches which restore `sysctl()` are circulating, though none have been merged. There appears to be some disagreement over whether removing `sysctl()` would truly break user-space applications or not. There are some uses of it in older C libraries, but, apparently, those libraries do the right thing when the attempt to use `sysctl()` fails, and applications operate normally. Linus has [asked](<https://lwn.net/Articles/204945/>) for an example of an application which truly breaks in the absence of `sysctl()`; none have been posted as of this writing. Interfaces which are not actually used on real systems are fair game for removal, so, unless somebody comes up with a a real-world problem soon, `sysctl()` will likely continue on its path out of the kernel.
