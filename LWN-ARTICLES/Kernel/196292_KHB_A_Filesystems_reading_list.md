---
title: "KHB: A Filesystems reading list"
url: https://lwn.net/Articles/196292/
date: "August 21, 2006"
category: Filesystems; Kernel Hackers Bookshelf
author: "August 21, 2006 This article was contributed by Valerie Aurora"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

August 21, 2006

This article was contributed by Valerie Aurora

We've all been there - you're wandering around a party at some Linux event clutching your drink and looking for someone to talk to, but everyone is having some obscure technical conversation full of unfamiliar jargon. Then, as you slide past a cluster of important-looking people, you overhear the word "superblock" and think, "Superblock, that's a file system thing... I read about file systems in operating systems class once." Gratefully, you join the conversation, only to discover while you know some of the terms - cylinder group, indirect block, inode - you're still unable to come up with stunning ripostes like, "Aha, but that's really just another version of soft updates, and it doesn't solve the nlinks problem." (Admiring silence ensues.) Now what? You want to be able to make witty remarks about the pros and cons of journaling while throwing back the last of your martini, but you don't know where to start.

Fortunately, you can get a decent grasp of modern file systems without reading a whole book on file systems. (I haven't yet read a book on file systems I would recommend, anyway.) After reading these file systems papers (or at least their abstracts), you'll be able to at least fake a working knowledge of file systems - as long as everyone is drinking and it's too loud to hear anyone clearly. Enjoy!

### The Basics

These papers are oldies but goodies. While the systems they describe are fairly obsolete and have been heavily improved since these initial descriptions, they make a good introduction to file systems structure and terminology.

[A Fast File System for UNIX ](<http://www.cs.berkeley.edu/~brewer/cs262/FFS.pdf>) by Marshall Kirk McKusick, William Joy, Samuel Leffler and Robert Fabry. This paper describes the first version of the original UNIX file system that was suitable for production use. It became known as FFS (Fast File System) or UFS (UNIX File System). The "fast" part of the name comes from the fact that the original UNIX file system maxed out at about 5% of disk bandwidth, whereas the first iteration of FFS could use about 50% - a huge improvement. This paper is absolutely foundational, as the majority of production UNIX file systems are FFS-style file systems. While some parts of this paper are obsolete (check out the section on rotational delay), it's a simple, readable explanation of basic file system architecture that you can refer back to time and again. Also, it's pretty fun to read a paper describing the first implementation of, for example, symbolic links for a UNIX file system.

For extra credit, you can read the original file system checker paper, [Fsck \- the UNIX file system check program](<http://citeseer.ist.psu.edu/rd/91293760%2C31351%2C1%2C0.25%2CDownload/http://citeseer.ist.psu.edu/cache/papers/cs/3106/ftp:zSzzSzftp.fwi.uva.nlzSzpubzSzcompzSzNetBSDzSzmisczSzlite2-docszSzsmmzSz03.fsck.pdf/mckusick94fsck.pdf>), by Marshall Kirk McKusick and T. J. Kowalski. It describes the major issues in checking and repairing file system metadata consistency. Improving fsck is a [hot topic in file systems](<http://lwn.net/Articles/190222/>) right now, so reading this paper might be worthwhile.

[Vnodes: An Architecture for Multiple File System Types in Sun UNIX](<http://www.solarisinternals.com/si/reading/vnode.pdf>) by Steve Kleiman. The original UNIX file system interface had been designed to support exactly one kind of file system. With the advent of FFS and other file systems, operating systems now needed to support several different file systems. Several solutions were proposed, but the dominant solution ended up being the VFS (Virtual File System) interface, first proposed and implemented by Sun. This paper explains the rationale behind VFS and vnodes.

[Design and Implementation of the Sun Network Filesystem](<http://citeseer.ist.psu.edu/rd/91293760%2C577908%2C1%2C0.25%2CDownload/http%3AqSqqSqwww.pdos.lcs.mit.eduqSq6.824qSqpapersqSqsandberg-nfs.pdf>) by Russel Sandberg, David Goldberg, Steve Kleiman, Dan Walsh, and Bob Lyon. Once upon a time (1985, specifically), people weren't really clear on why you would want a network file system (as opposed to, for example, a network disk or copying around files via rcp). This paper explains the needs and requirements that resulted in the invention of NFS, the network file system everyone loves to hate but uses all the time anyway. It also discusses the design of the VFS. A fun quote from the paper: ""One of the advantages of the NFS was immediately obvious: as the df output below shows, a diskless workstation can have access to more than a Gigabyte of disk!""

### Slaying the fsck dragon

One of the major problems in file systems is keeping the on-disk data consistent in the event that a file system is interrupted in the middle of update (for example, if the system loses power). Original FFS solved this problem by running fsck on the file system after a crash or other unclean unmount, but this took a really long time and could lose data. Many smart people thought about this problem and came up with four major approaches: journaling, log-structured file systems, soft updates, and copy-on-write. Each method provided a way of quickly recovering the file system after a crash. The most popular approach was journaling, since it was both relatively simple and easy to "bolt-on" to existing FFS-style file systems. 

Journaling file systems solve the fsck problem by first writing an entry describing an update to the file system to a on-disk journal - a record of file system operations. Once the journal entry is complete, the main file system is updated; if the operation is interrupted, the journal entry is replayed on the next mount, completing any half-finished operations in progress at the time of the crash. Most production file systems (including ext3, XFS, VxFS, logging UFS, and reiserfs) use journaling to avoid fsck after a crash. No canonical journaling paper exists outside the database literature (from whence the idea was lifted wholesale), but [Journaling the Linux ext2fs Filesystem](<http://citeseer.ist.psu.edu/rd/75103696%2C288237%2C1%2C0.25%2CDownload/http://citeseer.ist.psu.edu/cache/papers/cs/12964/http:zSzzSzwww.skylab.orgzSz%7EsabrezSzoszSzS3FileSystemszSzjournal-design.pdf/tweedie98journaling.pdf>) by Stephen Tweedie is a good choice for learning both journaling techniques in general and the details of ext3 in particular.

[The Design and Implementation of a Log-Structured File System](<http://citeseer.ist.psu.edu/rd/91293760%2C9977%2C1%2C0.25%2CDownload/http://citeseer.ist.psu.edu/cache/papers/cs/7286/http:zSzzSzguir.cs.berkeley.eduzSzprojectszSzosprelimszSzpaperszSzlfsSOSP91.pdf/rosenblum91design.pdf>) by Mendel Rosenblum and John K. Ousterhout. Journaling file systems have to write each operation to disk twice: once in the log, and once in the final location. What would happen if we only wrote the data to disk once - in the journal? While the log-structured architecture was an electrifying new idea, it ultimately turned out to be impractical for production use, despite the concerted efforts of many computer science researchers. Today, no major production file system is log-structured. (Note that a _log-structured_ file system is not the same as a _logging_ file system - logging is another name for journaling.) 

If you're looking for cocktail party _gossip_ , Margot Seltzer and several colleagues published papers critiquing and comparing log-structured file systems to variations of FFS-style file systems, in which LFS usually came out rather the worse for the wear. This led to a semi-famous flame war in the form of web pages, [archived here.](<http://www.eecs.harvard.edu/margo/papers/usenix95-lfs/supplement/>)

[Soft Updates: A Technique for Eliminating Most Synchronous Writes in the Fast Filesystem](<http://www.usenix.org/events/usenix99/full_papers/mckusick/mckusick.pdf>) by Marshall Kirk McKusick and Greg Ganger. Soft updates carefully orders writes to a file system such that in the event of a crash, the only inconsistencies are relatively harmless ones - leaked blocks and inodes. After a crash, the file system is mounted immediately and fsck runs in the background. The performance of soft updates is excellent, but the complexity is very high - as in, soft updates has been implemented only once (on BSD) to my knowledge. Personally, it took me about 5 years to thoroughly understand soft updates and I haven't met anyone other than the authors who claimed to understand it well enough to implement it. The paper is pretty understandable up to about page 5, at which point your head will explode. Don't feel bad about this, it happens to everyone.

[File System Design for an NFS File Server Appliance](<http://www.netapp.com/library/tr/3002.pdf>) by Dave Hitz, James Lau, and Michael Malcom. This paper describes the file system used inside NetApp file servers, Write-Anywhere File Layout (WAFL), as of 1994 (it's been improved in many ways since then). WAFL was the first major use of a copy-on-write file system - one in which "live" (in use) metadata is never overwritten in place but copied elsewhere on disk. Once a consistent set of updates has been written to disk, the "superblock" is re-written to point to the new set of metadata. Copy-on-write has an interesting set of trade-offs all its own, but has been implemented in a production file system twice now; [Solaris's ZFS](<http://www.opensolaris.org/os/community/zfs/>) is also a copy-on-write file system.

### File system performance

Each of these papers focuses on file system performance, but also introduces more than one interesting idea and makes a good starting point for exploring several areas of file system design and implementation.

[Extent-like Performance from a UNIX File System](<http://citeseer.ist.psu.edu/rd/0%2C23867%2C1%2C0.25%2CDownload/http%3AqSqqSqficus-www.cs.ucla.eduqSqproject-membersqSqreiherqSq..qSq..qSqclassesqSq239_2.spring96qSqpapersqSqextent_like_filesystem_performance.ps>) by Larry McVoy and Steve Kleiman. This 1991 paper describes optimizations to FFS that doubled file system bandwidth for sequential I/O workloads. While the optimizations described in this paper are considered old hat these days (ever heard of readahead?), it's a good introduction to file system performance.

### Sidebar: Where are they now?

You might have recognized some of the names in the author lists of the papers in this article - and chances are, you aren't recognizing their names because of their file system work. What else did these people do? Here's a totally non-scientific selection.

  * Bill Joy - co-founded Sun Microsystems 
  * Larry McVoy - wrote BitKeeper, co-founded BitMover 
  * Steve Kleiman - CTO, Network Appliance 
  * Mendel Rosenblum - co-founder, VMWare 
  * John Ousterhout - wrote Tcl/Tk, co-founded several companies 
  * Margot Seltzer - co-founder, Sleepycat Software 
  * Dave Hitz - co-founder, Network Appliance 
Obviously, anyone wanting to found a successful company and make millions of dollars should consider writing a file system first.   
---  
[Scalability in the XFS File System](<http://oss.sgi.com/projects/xfs/papers/xfs_usenix/index.html>) by Adam Sweeney, Doug Doucette, Wei Hu, Curtis Anderson, Mike Nishimoto, and Geoff Peck. This paper describes the motivation and implementation of XFS, a 64-bit file system using extents, B+ trees, dynamically allocated inodes, and journaling. XFS is not by any means an FFS-style file system and reading this paper will give you the basics on most extent-based file systems. It also describes quite a few useful optimizations for avoiding fragmentation, scaling to multiple threads, and the like.

[The Utility of File Names](<http://www.eecs.harvard.edu/~syrah/papers/tr-05-03/tr-05-03.pdf>) by Daniel Ellard, Jonathan Ledlie, and Margot Seltzer. File system performance and on-disk layout can be vastly improved if the file system can predict (with reasonable accuracy) the size and access pattern of a file before it writes it to disk. The obvious solution is to add a new set of file system interfaces allowing the application to give explicit hints about the size and properties of a new file. Unfortunately, the history of file systems is littered with unused per-file interfaces like this (how often do you set the noatime flag on a file?). However, it turns out that applications are already giving these hints - in the form of file names, permissions, and other per-file properties. This paper is the first in a series demonstrating that a file system can make useful predictions about the future of a file based on the file name and other properties.

### Further reading and acknowledgments

If you are interested in learning more about file systems, check out the [Linux file systems wiki](<http://linuxfs.pbwiki.com>), especially the [reading list](<http://linuxfs.pbwiki.com/ReadingList>). If you have a good file systems paper or book, please add it to the list, which is publicly editable (look for the password on the front page of the wiki). Note that I will ignore any comments of the form "You should have included paper XYZ!" unless it is also added to the reading list on the file systems wiki - WITH a short summary of the paper. With any luck, we'll have a fairly complete list of Linux file systems papers in the next few days.

~~If you are interested in working on file systems, or any other area of systems programming, you should contact the author at val dot henson at gmail dot com.~~

Thanks to Nikita Danilov, Zach Brown, and Kristen Accardi for paper suggestions and encouragement to write this article. Thanks to Theodore Y. Ts'o for actually saying something very similar to the stunning riposte in the first paragraph (which was, by the way, a completely accurate and very incisive criticism of what I was working on at the moment).
