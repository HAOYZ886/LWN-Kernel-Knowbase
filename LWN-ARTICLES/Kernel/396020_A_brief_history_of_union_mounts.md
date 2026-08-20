---
title: A brief history of union mounts
url: https://lwn.net/Articles/396020/
date: "July 14, 2010"
category: "Filesystems-Union; Union mounts"
author: "July 14, 2010 This article was contributed by Valerie Aurora"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

July 14, 2010

This article was contributed by Valerie Aurora

Several weeks ago, I mentioned on my blog that I planned [to move out of programming](<http://valerieaurora.wordpress.com/2010/06/14/moving-out-of-san-francisco-and-programming/>) in the near future. A few days later I received this email from a kernel hacker friend: 

At first, I thought we were losing a great hacker... But then I read on your blog: "Don't worry, I'm going to get union mounts into mainline before I change careers," and I realized this means you'll be with us for a few years yet! :) 

How long has union mounts existed without going into the mainline Linux kernel? Well, to put it in a human perspective, if you'd been born the same year as the first Linux implementation of union mounts, you'd be writing your college application essays right now. Werner Almsberger began work on the Inheriting File System, one of the early ancestors of Linux union mounts, in 1993 - 17 years ago! 

#### Background

A union mount does the opposite of a normal mount: Instead of hiding the namespace of the file system covered by the new mount, it shows a combination of the namespaces of the unioned file systems. Some use cases include a writable live CD/DVD-based system (without a complicated mess of symbolic links, bind mounts, and writable directories), and a shared base file system used by multiple clients. For an extremely detailed review of unioning file systems in general, see the LWN series: 

  * [Unioning file systems: Architecture, features, and design](<http://lwn.net/Articles/324291/>)

  * [Unioning file systems: Union directories and mounts](<http://lwn.net/Articles/325369/>)

  * [Unioning file systems: unionfs and aufs](<http://lwn.net/Articles/327738/>) 

This article will provide a high-level overview of various implementations of union mounts from the original 1993 Inheriting File System through the present day VFS-based union mount implementation and plans for near-term development. We deliberately leave aside unionfs, aufs, and other non-VFS implementations of unioning, in large part because the probability of merging a non-VFS unioning file system into mainline appears to be even lower than that of a VFS-based solution. 

#### `readdir()` redux

Throughout this article, we will place special emphasis on the evolution of `readdir()`, since historically it has been the greatest stumbling block for any implementation of union mounts. A summary from the first article in the LWN unioning file systems series: 

One of the great tragedies of the UNIX file system interface is the enshrinement of `readdir()`, `telldir()`, `seekdir()`, etc. family in the POSIX standard. An application may begin reading directory entries and pause at any time, restarting later from the "same" place in the directory. The kernel must give out 32-bit magic values which allow it to restart the `readdir()` from the point where it last stopped. Originally, this was implemented the same way as positions in a file: the directory entries were stored sequentially in a file and the number returned was the offset of the next directory entry from the beginning of the directory. Newer file systems use more complex schemes and the value returned is no longer a simple offset. To support `readdir()`, a unioning file system must merge the entries from lower file systems, remove duplicates and whiteouts, and create some sort of stable mapping that allows it to resume `readdir()` correctly. Support from userspace libraries can make this easier by caching the results in user memory. 

#### Union mounts development time line

As mentioned earlier, one of the first implementations of a unioning was the [Inheriting File System](<http://icapeople.epfl.ch/almesber/ifs.html>). In a pattern to be repeated by many future developers, Werner quickly became disenchanted with the complexity of the implementation of IFS and stopped working on it, suggesting that future developers try a mixed user/kernel implementation instead: 

Well, I completed it to the point where it was a nice proof of concept, but still with problems (leaks inodes, probably has a few races left, was also a bit too liberal with locking, etc.). 

Then I looked back at what I did and was disgusted by its complexity. So I decided that, before I might even consider proposing inclusion into the mainstream kernel, I'd have to see how much poorer (performance-wise) a user-space implementation would be. I did some initial hacking on NFS until I convinced myself that userfs might be the better approach. Unfortunately, I never found the time to work on that. 

Many other kernel developers agreed with Werner. One of Linus Torvalds' [earliest recorded NAKs](<http://lkml.indiana.edu/hypermail/linux/kernel/9603.1/0162.html>) of a kernel-based union file system came in 1996: 

While at USENIX, I saw the _correct_ way to do a union FS. It was done as a pre-loaded shared library, and because of that it was a lot more flexible than any kernel implementation would ever be [...] After having seen that, I don't think I necessarily would even want a kernel implementation. It simply was so much better done in user space. 

In 1998, Werner updated his IFS page to suggest working on a unioning file system as a good academic research topic: 

Sounds like a very nice master's thesis topic for some good Linux hacker ;-) [...] So far nobody has taken the challenge. So, if you're an aspiring kernel hacker, aren't afraid of complexity, have a lot of time, and are looking for an interesting but useful project, you may just have found it :-) 

Around 2003 - 2004, Jan Blunck took up the gauntlet Werner threw down and began [working on union mounts](<http://www.linuxtag.org/2005/freecongress-details-talkid-155.html>) for his thesis. The union mount implementation Jan produced lay dormant until 2007, when discussion about merging [unionfs](<http://en.wikipedia.org/wiki/UnionFS>) into mainline triggered [renewed interest](<http://marc.info/?l=linux-kernel&m=117524884207813>) in a VFS-based version of unioning. At that point, Bharata B. Rao took the lead and began working with Jan Blunck on a new version of union mounts. Bharata and Jan posted several versions in 2007. 

The [first version posted](<http://marc.info/?l=linux-fsdevel&m=117681527820133>) in April 2007 used Jan's original strategy of keeping two pointers in the dentry for each directory, one pointing to the directory below this dentry's in the union stack, and one to the dentry of the topmost directory. The drawback to this implementation is that each file system can only be in one union stack at a time, since dentries are shared between all mounts of the same underlying file system. 

The [second version posted](<http://marc.info/?l=linux-fsdevel&m=117913503200362>) in May 2007 implemented yet another minor variation on in-kernel `readdir()`, this time using per file pointer cookies. From the patch set's documentation: 

When two processes issue `readdir()`/`getdents()` call on the same unioned directory, both of them would be referring to the same dentries via their file structures. So it becomes necessary to maintain rdstate separately for these two instances. This is achieved by using a cookie variable in the rdstate. Each of these rdstate instances would get a different cookie thereby differentiating them. 

In June 2007, Bharata and Jan [posted a third version](<http://marc.info/?l=linux-fsdevel&m=118231827024394>) with an important and novel change to the way union stacks are formed. They replaced the in-dentry links to the topmost and lower directories with an external structure of pointers to (vfsmount, dentry) pairs. For the first time, a file system could be part of more than one union mount. From the patch set's documentation: 

In this new approach, the way union stack is built and traversed has been changed. Instead of dentry-to-dentry links forming the stack between different layers, we now have (vfsmount, dentry) pairs as the building blocks of the union stack. Since this (vfsmount, dentry) combination is unique across all namespaces, we should be able to maintain the union stack sanely even if the filesystem is union mounted privately in different namespaces or if it appears under different mounts due to various types of bind mounts. 

In July 2007, Jan [posted a fourth version](<http://marc.info/?l=linux-fsdevel&m=118581230916874>) with some relatively minor changes to the way whiteouts were implemented, among a few other things. Jan says, ""I'm able to compile the kernel with this patches applied on a 3 layer union mount with the [separate] layers bind mounted to different locations. I haven't done any performance tests since I think there is a more important topic ahead: better `readdir()` support."" 

In December 2007, Bharata B. Rao [posted a fifth version](<http://marc.info/?l=linux-fsdevel&m=119686559720586>) that implemented another in-kernel version of `readdir()`: 

In this approach, the cached dirents are given offsets in the form of linearly increasing indices/cookies (like 0, 1, 2,...). This helps us to uniformly define offsets across all the directories of the union irrespective of the type of filesystem involved. Also this is needed to define a seek behaviour on the union mounted directory. This cache is stored as part of the struct file of the topmost directory of the union and will remain as long as the directory is kept open. 

However, this approach had multiple problems, including excessive use of kernel memory to cache directory entries and to keep the mapping of indices to dentries. 

`readdir()` continued to be a stumbling block, and union mounts development slowed down for most of 2008. In April 2008, Nagabhushan BS implemented and [posted](<http://marc.info/?l=linux-fsdevel&m=120947680420674>) a version of union mounts with most of the `readdir()` logic moved to glibc. ""I went through Bharata's RFC post on glibc based Union Mount readdir solution (<http://lkml.org/lkml/2008/3/11/34>) and have come up with patches against glibc to implement the same."" 

However, moving the complexity to user space wasn't the panacea everyone had hoped for. The glibc maintainers had many objections, the kernel interface was an obvious kludge (returning whiteouts for "." to signal a unioned directory), and no one could figure out how to handle NFS sanely. 

In November 2008, Miklos Szeredi [posted](<http://marc.info/?l=linux-fsdevel&m=122787012403963>) a simplified version of union mounts that implemented `readdir()` in the kernel. 

The directory entries are read starting from the top layer and they are maintained in a cache. Subsequently when the entries from the bottom layers of the union stack are read they are checked for duplicates (in the cache) before being passed out to the user space. There can be multiple calls to readdir/getdents routines for reading the entries of a single directory. But union directory cache is not maintained across these calls. Instead for every call, the previously read entries are re-read into the cache and newly read entries are compared against these for duplicates before being they are returned to user space. 

This implementation only worked for file systems that return a simple increasing offset in the d_off field for `readdir()`. So ext2 worked, but any file system with a modern directory hashing scheme did not. 

In early 2009, I started to get interested in union mounts. I talked to several groups inside Red Hat and asked them what they needed most from file systems. I heard the same story over and over: ""We really really need a unioning file system, but for some reason no one at Red Hat will support unionfs..."" I did some research on the available implementations and decided to go to work on Jan Blunck's union mount patch set. 

In May 2009, Jan Blunck and I [posted](<http://marc.info/?l=linux-fsdevel&m=124266382908872>) a version of union mounts that implemented in-kernel `readdir()` using a new concept: the fallthru directory entry. The basic idea is that the first time `readdir()` is called on a directory, the visible directory entries from all the underlying directories are copied up to the topmost directory as fallthru directory entries. This eliminated all the problems I knew of in previous `readdir()` implementations, but required the topmost file system to always be read-write. This implementation also was limited to only two layers: one read-only file system overlaid with one read-write file system because we were concerned with lock ordering problems. 

In October 2009, I [posted](<http://marc.info/?l=linux-fsdevel&m=125615355718507>) a version of union mounts that implemented some of the more difficult system calls, such as `truncate()`, `pivot_root()`, and `rename()`. However, implementing `chmod()` and other system calls that modified files without opening them turned out to be fairly difficult with the current code base. We thought the hard part was copying up file data in `open()`, `rename`, and `link()`, but it turned out they were somewhat easier to implement because they already looked up the parent directory of the file to be altered. For union mounts, we need the parent directory's `dentry` and `vfsmount` in order to create a new version of the file in the topmost level of the union file system if necessary. `open()`, `rename`, and `link()` also needed the parent directory in order to create new directory entries, so we just reused the parent in the union mount in-kernel copyup code. But system calls like `chmod()` that only alter existing files did not bother to lookup the parent directory's path, only the target. Regretfully, I decided to start on a major rewrite. 

In March 2010, I [posted](<http://marc.info/?l=linux-fsdevel&m=126756790019402>) a rewrite of the pathname lookup mechanism for union mounts, taking into account Al Viro's recent VFS cleanups and removing a great deal of unnecessary code duplication. 

In May 2010, I [ posted](<http://marc.info/?l=linux-fsdevel&m=127292910632750>) the first version of union mounts that implemented nearly all file related system calls correctly. The four exceptions were `fchmod()`, `fchown()`, `fsetxattr()`, and `futimensat()`, which will fail on read-only file descriptors. (UNIX is full of surprises; none of the VFS experts I talked to knew that these system calls would succeed on a read-only fd.) 

The central primitive in this version is a function called `user_path_nd()`. It is a combination of `user_path()`, which looks up a pathname and returns the corresponding dentry and vfsmount, and `user_path_parent()`, which looks up the parent directory of the file or directory given by the pathname and returns the `struct nameidata` for the parent. (`struct nameidata` is too complex to describe in full here, but suffice to say it is usually needed to create an entry in a directory.) `user_path_nd()` returns both the parent's `nameidata` and the child's path. Once we have both these pieces of information, we can do an in-kernel copyup of a file in `chmod()` or any other system call that modifies a file. Unfortunately, `user_path_nd()` is also the weakest point in this version of union mounts: it's racy, inefficient, and copies up files even if the system call fails. 

The day after I posted that version, I flew to North Carolina for a long-anticipated in-person code review with Al Viro. We spent three days in his office painfully reviewing the entire union mount implementation. Al immediately figured out how to delete a third of the code I'd spent the last year carefully massaging, and then outlined how to rewrite the other two-thirds of the code more elegantly, including `user_path_nd()`. As a result of this code review marathon, Linux has a feature-complete implementation of union mounts that has undergone a full code review by the Linux VFS maintainer for the first time. Of course, the resulting todo list is long and complex, and some problems may turn out to be insoluble, but it's an important step forward. 

The biggest design change Al suggested was to move the head of the union stack back into the dentry, while keeping the rest of the union stack in a singly linked list of `struct union_dir`'s allocated external to the dentries for the read-only parts of the union stack. This combines the speed and elegance of Jan Blunck's original design using in-dentry pointers to the union stack, with the flexibility of Bharata B. Rao's `(vfsmount, dentry)` pairs, which allow file systems to be part of many read-only layers. This change removed the entire union stack hash table and the associated lookup logic and shrunk the `union_dir` struct from 7 members to 2. I [posted](<http://marc.info/?l=linux-fsdevel&m=127662792813470>) this hybrid linked list version on June 15, 2010. 

Most recently, on June 25th, 2010, I [posted](<http://marc.info/?l=linux-fsdevel&m=127749275818612>) a version that implemented submounts in the read-only layers, as well as allowed more than two read-only layers again. Then I went on a two week vacation - the longest vacation I've had since I started working on union mounts - and tried to forget everything I knew about it. 

#### Future Work

The next step is to implement the remainder of Al Viro's review comments. The last big-ticket item is rewriting `user_path_nd()` and the in-kernel file copyup boilerplate. After that, it's back for another round of code review from Al and the other VFS maintainers. The 2010 Linux Storage and File Systems workshop is in early August. With luck we can hash out any remaining architectural problems face-to-face at the workshop and possibly merge union mounts into mainline before it's old enough to vote. Or it might languish for another 17 years outside the kernel. Such are the vicissitudes of Linux kernel development. 

Acknowledgments: I want to extend special thanks to the following people: Kevin Roderick, who provided moral support, Tim Bowen, who gave me a free day at the [Spoke6 co-working space](<http://www.spoke6.com>) while I worked on this article, and, of course, Jake Edge, whose editorial suggestions were, as usual, right on.
