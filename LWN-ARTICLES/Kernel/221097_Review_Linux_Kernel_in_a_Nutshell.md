---
title: "Review: Linux Kernel in a Nutshell"
url: https://lwn.net/Articles/221097/
date: "February 7, 2007"
category: Books
author: 
---

Once upon a time, the ability to download, compile, and install a new kernel was a vital skill for any Linux system administrator. That skill is less in demand now; the kernels shipped with most distributions tend to be adequate for most needs. Still, there comes a time, even for those who do not hack on the kernel itself, when a system needs a custom kernel. Many system administration books devote a bit of space to this task, but they ![](https://static.lwn.net/images/ns/kernel/kernel-nutshell.gif) tend to pass over it fairly quickly. Configuring, building, and installing a kernel remains a relatively dark art for many. 

Kernel hacker Greg Kroah-Hartman decided to do something about it; the result is _Linux Kernel in a Nutshell_ , published by O'Reilly. By the standards of other kernel books from that publisher, this is a thin volume indeed: just over 180 pages, including the index. But it is packed with information that should be useful to just about anybody who has to deal with the kernels on their systems. 

The early chapters cover some of the basics: what tools are required, where to get the kernel source, etc. There is a chapter on the various ways of configuring a kernel. Your editor remembers the days of configuring kernels by stepping through the entire "make config" process; it's nice to see Greg recommending against that approach now. The build process is discussed, as are the necessary steps for installing the kernel once it's built. 

The second major part of the book discusses customizations - in particular, enabling support for a device. The process for determining which driver should be enabled for a specific device is distressingly hairy; it involves listing out the PCI bus configuration, digging through sysfs, then trying to find a match in the kernel source. It's not for nothing that Greg says: 

The easiest way to figure out which driver controls a new device is to build all of the different drivers of that type in the kernel source tree as modules, and let the udev startup process match the driver to the device. 

As they say, there really should be a better way. But one can't fault Greg for telling it like it is. 

Next there is a set of "kernel configuration recipes" for enabling specific behavior. The advice here is terse, sometimes to a fault. The discussion on enabling kernel preemption, for example, could have benefited from a mention of the reliability concerns which have kept most distributors from turning preemption on. Similarly, it talks about how to enable SELinux with no mention of the need for an accompanying policy loaded from user space. The audience for this book seems likely to include quite a few people from the "know just enough to hurt themselves" population; a few more hints might have proved most helpful to those readers. 

The final section, making up almost half of the book, is devoted to reference material. There is an extensive list of kernel command line parameters and what they do - though the treatment is, once again, terse. There is a useful chapter on the various `make` targets and options for the kernel; somehow your editor had managed to avoid learning about `make randconfig` until now. There is also a reference chapter for configuration options. This chapter is incomplete, however, and the options do not appear to be listed in any particular order. 

Minor grumbles aside, there is value in this book's conciseness. When faced with a question about kernel configuring, building, or booting, this book is likely to yield an answer without forcing the reader to search for a needle in an 800-page haystack. It covers an area which was very much in need of some improved documentation; it is also reasonably up to date, having been written for the 2.6.18 kernel. Happily, Greg has [made the book available online](<http://kroah.com/lkn/>). Overall, _Linux Kernel in a Nutshell_ is a more than welcome addition to your editor's bookshelf.
