---
title: Killing off /dev/kmem
url: https://lwn.net/Articles/851531/
date: "April 5, 2021"
category: devkmem
author: "By Jonathan Corbet April 5, 2021"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 5, 2021

The recent [proposal](<https://lwn.net/ml/linux-kernel/20210319143452.25948-1-david@redhat.com/>) from David Hildenbrand to remove support for the `/dev/kmem` special file has not sparked a lot of discussion. Perhaps that is because today's youngsters, lacking an understanding of history, may be wondering what that file is in the first place and, thus, be unclear on why it may matter. Chances are that `/dev/kmem` will not be missed, but in passing it takes away a venerable part of the Unix kernel interface. 

`/dev/kmem` provides access to the kernel's address space; it can be read from or written to like an ordinary file, or mapped into a process's address space. Needless to say, there are some mild security implications arising from providing that sort of access; even read access to this file is generally enough to expose credentials and allow an attacker to take over a system. As a result, protections on `/dev/kmem` have always tended to be restrictive, but it remains the sort of open back door into the kernel that makes anybody who worries about security worry even more. 

It is a rare Linux system that enables `/dev/kmem` now. As of the 2.6.26 kernel release in July 2008, the kernel only implements this special file if the `CONFIG_DEVKMEM` configuration option is enabled. One will have to look long and hard for a distributor that enables this option in 2021; most of them disabled it many years ago. So its disappearance from the kernel is unlikely to create much discomfort. 

It's worth noting that Linux systems still support `/dev/mem` (without the "`k`"), which once provided similar access to _all_ of the memory in the system. It has long been restricted to I/O memory; system RAM is off limits. The occasional user-space device driver still needs `/dev/mem` to function, but it's otherwise unused. 

One may well wonder why a dangerous interface like `/dev/kmem` existed in the first place. The kernel goes out of its way to hide its memory from the rest of the system; creating a special file to circumvent that hiding seems like a step in the wrong direction. The answer, in short, is that once there was no other way to get many types of information out of the kernel. 

As an example, consider the "load average" numbers printed by tools like `top`, `uptime`, or `w`; they indicate the average length of the CPU run queues over periods of one, five, and 15 minutes. In the distant past, when computers were scarce and it was common to run many tasks on the same machine, jobs that were not time-critical would often consult the load average and defer their work if it was too high. It was the sort of courtesy that was much appreciated by the other users of the machine, of which there may have been dozens. 

But how does one determine the current load average? Unix kernels have maintained those statistics for decades, but they originally kept that information to themselves. User-space code that wanted to know this number would have to do the following: 

  * Read the symbol table from the executable image of the current kernel to determine the location of the `avenrun` array. 
  * Open `/dev/kmem` and seek to that location. 
  * Read the `avenrun` array into a user-space buffer. 

Code from that era can be hard to find, but the truly masochistic can wade through what must be one of the deeper circles of `#ifdef` hell to find an implementation toward the bottom of [this version of `getloadavg()`](<https://opensource.apple.com/source/gnumake/gnumake-131/getloadavg.c.auto.html>) from an early GNU `make` release. In a current Linux system, instead, all that is needed is to read a line from `/proc/loadavg`. 

This kind of grubbing around in kernel memory was not limited to the load-average array. Tools with more complex information requirements also had to dig around in `/dev/kmem`; see, for example, [the 2.9BSD implementation of `ps`](<https://www.tuhs.org/cgi-bin/utree.pl?file=2.9BSD/usr/src/cmd/ps/ps.c>). That was just how things were done in those days. 

Rooting through the kernel's memory for information about the system has a number of problems beyond the need to implement `/dev/kmem`. Changes to the kernel could break user space in surprising ways. Multiple reads were often needed to get a complete picture, but that picture could change while the reads were taking place, leading to more surprises. The move away from `/dev/kmem` and toward well-defined kernel interfaces, such as `/proc`, sysfs, and various system calls, has cleaned this situation up — and made it possible to disable `/dev/kmem`. 

Now, it seems that `/dev/kmem` will go away entirely. Linus Torvalds [said](<https://lwn.net/ml/linux-kernel/CAHk-=wg1HTbXkjdMYA4zADEiE8HwpZ0=uWy0ujZTJYVT-KCehQ@mail.gmail.com/>) that he would ""happily do this for the next merge window"", but he wanted confirmation that distributors are, indeed, not enabling it now. There have been a few responses for specific distributions, but nobody has said that `/dev/kmem` is still in use anywhere. If there _are_ users of this interface out there, they will want to make their existence known in the near future. Failing that, this back door into kernel memory will soon be removed entirely — but, then, your editor [once predicted](<https://lwn.net/Articles/147901/>) that it would be removed for 2.6.14, so one never knows.
