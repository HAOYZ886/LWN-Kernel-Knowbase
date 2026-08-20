---
title: One billion files on Linux
url: https://lwn.net/Articles/400629/
date: "August 18, 2010"
category: Filesystems; Scalability
author: "By Jonathan Corbet August 18, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 18, 2010

What happens if you try to put one billion files onto a Linux filesystem? One might see this as an academic sort of question; even the most enthusiastic music downloader will have to work a while to collect that much data. It would require over 30,000 (clean) kernel trees to add up to a billion files. Even contemporary desktop systems, which often seem to be quite adept at the creation of vast numbers of small files, would be hard put to make a billion of them. But, Ric Wheeler says, this is a problem we need to be thinking about now, or we will not be able to scale up to tomorrow's storage systems. His LinuxCon talk used the billion-file workload as a way to investigate the scalability of the Linux storage stack. 

One's first thought, when faced with the prospect of handling one billion files, might be to look for workarounds. Rather than shoveling all of those files into a single filesystem, why not spread them out across a series of smaller filesystems? The problems with that approach are that (1) it limits the kernel's ability to optimize head seeks and such, reducing performance, and (2) it forces developers (or administrators) to deal with the hassles involved in actually distributing the files. Inevitably things will get out of balance, forcing things to be redistributed in the future. 

Another possibility is to use a database rather than the filesystem. But filesystems are familiar to developers and users, and they come with the operating system from the outset. Filesystems also are better at handling partial failure; databases, instead, tend to be all-or-nothing affairs. 

[![\[Ric Wheeler\]](https://static.lwn.net/images/conf/2010/linuxcon/RicWheeler-sm.jpg)](<https://lwn.net/Articles/400640/>) If one wanted to experiment with a billion-file filesystem, how would one come up with hardware which is up to the task? The most obvious way at the moment is with external disk arrays. These boxes feature non-volatile caching and a hierarchy of storage technologies. They are often quite fast at streaming data, but random access may be fast or slow, depending on where the data of interest is stored. They cost $20,000 and up. 

With regard to solid-state storage, Ric noted only that 1Tb still costs a good $1000. So rotating media is likely to be with us for a while. 

What if you wanted to put together a 100Tb array on your own? They did it at Red Hat; the system involved four expansion shelves holding 64 2Tb drives. It cost over $30,000, and was, Ric said, a generally bad idea. Anybody wanting a big storage array will be well advised to just go out and buy one. 

The filesystem life cycle, according to Ric, starts with a mkfs operation. The filesystem is filled, iterated over in various ways, and an occasional fsck run is required. At some point in the future, the files are removed. Ric put up a series of plots showing how ext3, ext4, XFS, and btrfs performed on each of those operations with a one-million-file filesystem. The results varied, with about the only consistent factor being that ext4 generally performs better than ext3. Ext3/4 are much slower than the others at creating filesystems, due to the need to create the static inode tables. On the other hand, the worst performers when creating 1 million files were ext3 and XFS. Everybody except ext3 performs reasonably well when running fsck - though btrfs shows room for some optimization. The big loser when it comes to removing those million files is XFS. 

To see the actual plots, have a look at [Ric's slides [PDF]](<http://events.linuxfoundation.org/slides/2010/linuxcon2010_wheeler.pdf>). 

It's one thing to put one million files into a filesystem, but what about one _billion_? Ric did this experiment on ext4, using the homebrew array described above. Creating the filesystem in the first place was not an exercise for the impatient; it took about four hours to run. Actually creating those one billion files, instead, took a full four days. Surprisingly, running fsck on this filesystem only took 2.5 hours - a real walk in the park. So, in other words, Linux can handle one billion files now. 

That said, there are some lessons that came out of this experience; they indicate where some of the problems are going to be. The first of these is that running fsck on an ext4 filesystem takes a _lot_ of memory: on a 70Tb filesystem with one billion files, 10GB of RAM was needed. That number goes up to 30GB when XFS is used, though, so things can get worse. The short conclusion: you can put a huge amount of storage onto a small server, but you'll not be able to run the filesystem checker on it. _That_ is a good limitation to know about ahead of time. 

Next lesson: XFS, for all of its strengths, struggles when faced with metadata-intensive workloads. There is work in progress to improve things in this area, but, for now, it will not perform as well as ext3 in such situations. 

According to Ric, running `ls` on a huge filesystem is "a bad idea"; iterating over that many files can generate a lot of I/O activity. When trying to look at that many files, you need to avoid running `stat()` on every one of them or trying to sort the whole list. Some filesystems can return the file type with the name in `readdir()` calls, eliminating the need to call `stat()` in many situations; that can help a lot in this case. 

In general, enumeration of files tends to be slow; we can do, at best, a few thousand files per second. That may seem like a lot of files, but, if the target is one billion files, it will take a very long time to get through the whole list. A related problem is backup and/or replication. That, too, will take a very long time, and it can badly affect the performance of other things running at the same time. That can be a problem because, given that a backup can take days, it really needs to be run on an operating, production system. Control groups and the I/O bandwidth controller can maybe help to preserve system performance in such situations. 

Finally, application developers must bear in mind that processes which run this long will invariably experience failures, sooner or later. So they will need to be designed with some sort of checkpoint and restart capability. We also need to do better about moving on quickly when I/O operations fail; lengthy retry operations can take a slow process and turn it into an interminable one. 

In other words, as things get bigger we will run into some scalability problems. There is nothing new in that revelation. We've always overcome those problems in the past, and should certainly be able to do so in the future. It's always better to think about these things before they become urgent problems, though, so talks like Ric's provide a valuable service to the community.
