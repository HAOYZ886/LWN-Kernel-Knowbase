---
title: "Reiser4 - the mammoth arrives"
url: https://lwn.net/Articles/14035/
date: "October 30, 2002"
category: "Filesystems-Reiser4"
author: 
---

One of the remaining issues to be resolved before the Halloween feature freeze is whether the Reiser4 filesystem will be included in the 2.5 kernel. This has been a hard question to answer, however, given that almost nobody had actually seen the Reiser4 source. That situation, at least, has come to an end with the [announcement](<https://lwn.net/Articles/14002/>) of the first public Reiser4 snapshot. 

Reiser4 is the latest incarnation of the ReiserFS filesystem. It is not simply an upgrade; Reiser4 has been redesigned and reimplemented from the beginning. It is a completely different filesystem than the ReiserFS (also known as "Reiser3") found in the 2.4 kernel; should it be included, the next stable kernel will contain both Reiser3 and Reiser4, as separate options. 

There is a fair amount of online information available on Reiser4, though some of it makes for a bit of a challenging read. [This lengthy document](<http://www.namesys.com/v4/v4.html>) provides discussion in depth of many of the Reiser4 features (not all of which are implemented yet), along with an explanation of Hans Reiser's long-term vision for filesystems, a polemic on free software, and some of the weirdest imagery to be found in software documentation anywhere. The document entitled [The Infrastructure for Security Attributes in Reiser4](<http://www.namesys.com/v4/fast_reiser4.html>) is actually a relatively straightforward discussion of many of the technical details behind the Reiser4 design, and might be a better starting point. 

For those wanting a shorter summary, here's a few of the features to be found in Reiser4: 

  * The filesystem maintains many of the basic features of Reiser3 - it is based on (mostly) balanced trees, with file data incorporated in the tree along with names. Reiser4 thus remains well suited to the handling of large numbers of very small files. 

  * It is smarter about block allocation and data placement. Block allocation is delayed until file data is actually written to disk, leading to more efficient layouts. On-disk layout is done with extents. The result of these optimizations is that the filesystem's read performance is greatly improved over Reiser3. 

  * "Wandering logfiles" take some techniques from log-structured filesystems to provide journaling without (always) writing data to the disk twice. In many cases, Reiser4 can write "journal" data to a disk block, then atomically swap the journal block into the file itself. The journaling code can overwrite or replace blocks, depending on which technique would provide better layout on the disk. 

  * Most filesystem semantics are implemented with plugins. The normal Unix directory behavior, for example, is implemented with the "Unix directory plugin." Plugins can be used to implement security features (access control lists and such), encryption, maintenance of audit trails, and no end of strange, non-POSIX semantics. Hans Reiser remains determined to implement a lot of interesting features in his filesystem, and plugins are the mechanism by which those features will be included. 

  * Reiser4 is heavily transaction-oriented, and is able to provide guarantees that operations will be performed atomically. Future plans call for the ability to perform multi-file operations in an atomic manner. 

  * The Reiser4 design includes a `reiser4()` system call ""to support applications that don't have to be fooled into thinking that they are using POSIX."" This system call will accept (and parse) command strings that can describe complex operations. The `reiser4()` system call is not implemented in the current snapshot. 

As an example of the sort of uses that the Reiser4 developers eventually would like to see, consider the classic Unix password file. Each line in the file describes one account, and contains several colon-separated fields with information like the account name, user and group IDs, the user's home directory and shell, etc. In Reiser4, each field in the password file would become a file in its own right; one could obtain the home directory of a given `user` via a path like: 

```
/etc/passwd/user/home
```

A special-purpose plugin would aggregate the various files, so that a process reading `/etc/passwd` would see the same information as always. But each field file could be protected differently; a user could have write access to the file describing his or full name, but not to the one containing the user ID value. 

In the Reiser4 vision, file attributes would also be stored as files. For a given `file`, something like `file/owner` would contain the UID of the user who owns that file. 

Needless to say, in the long-term Reiser vision, Linux systems will behave rather differently than they do now. In the shorter term, Reiser4 promises a high-performance journaling filesystem with highly efficient handling of small files and a plugin architecture which encourages experiments with interesting new semantics. 

Will it be merged? The Reiser4 team plans to submit a patch for merging at the last second, sometime before midnight on Halloween. Some developers have argued that it is too late to propose a major new feature that nobody has had a chance to look at. Hans [feels this is inappropriate](<https://lwn.net/Articles/14152/>): 

I'm the last straggler coming back from the hunt, and I've got what looks like it might be a wooly mammoth on my shoulders, and my tribesmen are complaining that I'm late for dinner. How about helping me by cutting down a tree for the roasting spit instead 

Linus has not offered any public opinions on the matter. The Reiser4 patch is apparently unintrusive, however, so there is probably no real reason not to include it.
