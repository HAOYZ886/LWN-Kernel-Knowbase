---
title: Looking at reiser4
url: https://lwn.net/Articles/99232/
date: "August 25, 2004"
category: "Filesystems-Reiser4; sys reiser4"
author: 
---

The reiser4 filesystem came one step closer to inclusion when it was added to [2.6.8.1-mm2](<https://lwn.net/Articles/98658/>). This filesystem was covered here [in July, 2003](<https://lwn.net/Articles/41652/>); those interested in a lengthy writeup with lots of details and weird artwork can find it at [namesys.com](<http://www.namesys.com/>). In short, reiser4's claims include very high performance, high-level transactional capability, enhanced security, and a flexible plugin architecture which should make it possible to do truly different and interesting things. 

Actually playing with reiser4 involves getting a recent -mm kernel (or downloading it separately and applying it to another kernel). The tools for building and checking reiser4 filesystems can be found [over here](<ftp://ftp.namesys.com/pub/reiser4progs/>). There is a shareable library ("libaal") which must be built first, followed by the "reiser4progs" package. If the reiser4progs configuration process tells you that you lack the proper version of libaal, it probably means you forgot to run `ldconfig` between the two steps. 

We ran some very simple tests using the only benchmark that really matters: working with the kernel source tree. The first step was to look at the simple usage of space; reiser4 claims to be more efficient in that regard. This table indicates how much space was used (in KB) in various points in the kernel build process: 

Filesystem | Space usage  
---|---  
Empty| New kernel tree| Built kernel tree  
reiser4 | 188 | 206,000 | 659,000  
ext3 | 32,800 | 271,000 | 727,000  
  
An empty ext3 filesystem has a fair amount of overhead (almost 33MB on a 2GB partition) that is not seen on reiser4; the reason is that reiser4 does not need to pre-allocate any inode tables. That saves some space; it also means that reiser4 filesystems will never run out of inodes. Reiser4 is also clearly more efficient in its file layout; an unbuilt kernel tree takes about 15% less space than on ext3. 

The next step was a set of highly unscientific timing tests involving various tasks: untarring a kernel, building that kernel, [grepping dirty words](<https://lwn.net/Articles/99240/>) out of the kernel source, and two `find` commands: one which tests on file names only, and one requiring a `stat()` of each file. The tests were run on some bleeding-edge hardware: an otherwise unused 4GB IDE disk on a dual Pentium-450 system. The filesystem was unmounted between tests to clear its pages out of the cache. Here's the results; two times are presented: elapsed and system. 

Filesystem | Test  
---|---  
Untar | Build | Grep | find (name) | find (stat)  
reiser4 | 67/41 | 1583/386 | 78/12 | 12.5/1.3 | 15.2/4.0  
ext3 | 55/24 | 1400/217 | 62/8 | 10.4/1.1 | 12.1/2.5  
  
Anybody who tries to draw any real conclusions from the above results should probably think again. That said, it would seem that reiser4's claim to being the fastest Linux filesystem remains unproven. Incidentally, here's another quote from the reiser4 configuration help text: 

If using a kernel made by a distro that thinks they are our competitor (sigh) rather than made by Linus, always check each release to make sure they have not turned this on to make us look slow as was done once in the past. 

This text describes a debugging option; that option was _not_ enabled for these tests. 

Meanwhile, the inclusion of reiser4 into -mm has, as desired, increased the number of developers looking at the code. Many of them are not entirely happy with what they see. The first problem is that reiser4 will fail horribly with 4K kernel stacks; it seems that quite a few large data structures are kept on the stack. The reiser4 hackers will be looking at reworking memory allocation to get around that particular problem. 

Rik van Riel was the first to [stumble across](<https://lwn.net/Articles/99255/>) the `sys_reiser4()` system call. The code to implement `sys_reiser4()` is present (and built) in -mm, but the actual call is not added to the system call table. A patch comes with the source to make that addition, however. 

According to the documentation: 

A new system call sys_reiser4() will be implemented to support applications that don't have to be fooled into thinking that they are using POSIX. Through this entry point a richer set of semantics will access the same files that are also accessible using POSIX calls.... Reiser4() will implement all features necessary to access ACLs as files/directories rather than as something neither file nor directory. These include opening and closing transactions, performing a sequence of I/Os in one system call, and accessing files without use of file descriptors (necessary for efficient small I/O). Reiser4 will use a syntax suitable for evolving into Reiser5() syntax with its set theoretic naming. 

This syntax, it seems, is implemented via a yacc-generated parser, which is duly stuffed into the kernel. As Rik notes, this approach is likely to be controversial, even before people start thinking about what the new operations actually do. 

Reiser4 blurs the distinction between files and directories as part of Hans Reiser's general view of how filesystems should be used. For example, extended attributes, according to Hans, should not exist in their own namespace; they should just look like more files. With the right plugins, it should also be possible to do things like treat a `tar` archive as a directory tree and move around within it. There are, it seems, some immediate problems with this idea. As Christoph Hellwig [pointed out](<https://lwn.net/Articles/99258/>), reiser4 allows an open with the `O_DIRECTORY` flag to succeed even if the target is not a directory. That defeats the use of `O_DIRECTORY` as a way of avoiding race conditions and security holes, and is unlikely to go over well. Al Viro [noted](<https://lwn.net/Articles/99260/>) some severe locking problems (leading to easy denial of service attacks) with the file-as-directory implementation as well. 

Reiser4, it seems, may have a bit of a rough road on its way into the kernel. Hans's [approach to PR](<https://lwn.net/Articles/99302/>) is unlikely to help in this regard, though it should be noted that Linus [likes](<https://lwn.net/Articles/99303/>) some of the reiser4 features. One hopes that reiser4 will get into the kernel eventually. It would surely be a mistake to believe that the optimal set of filesystem semantics has been achieved. The reiser4 project is arguably the place where the most thinking is happening about where filesystems should go in the future. If Linux is unwilling to host the results of that work (after the obvious problems are fixed), it may eventually find itself trying to catch up with some other kernel which proves to be more accepting.
