---
title: Merging kdb and kgdb
url: https://lwn.net/Articles/374633/
date: "February 17, 2010"
category: "Debugging; Development tools-kgdb; kdb; kgdb"
author: "By Jake Edge February 17, 2010"
---

By **Jake Edge**  
February 17, 2010

It was something of a surprise when Linus Torvalds [merged](<http://lwn.net/Articles/278678/>) kgdb--a stub to talk to the gdb debugger--back in the 2.6.26 merge window, because of his well-known [disdain](<http://lwn.net/2000/0914/a/lt-debugger.php3>) for kernel debuggers. But there is another kernel debugging solution that has long been out of the mainline: kdb. Jason Wessel has [proposed](<http://lwn.net/Articles/374203/>) merging the two solutions by reworking kgdb to use the "kdb shell" underneath, which would lead to both solutions being available for kernel hackers. 

The two debuggers serve different purposes, with kdb having much less functionality, but they both have uses. Kgdb allows source-level debugging using gdb over a serial line, but that requires a separate system. For systems where it is painful or impractical to set up a serial connection, kdb may provide enough capability to debug a problem. In addition, things like kernel modesetting (KMS) allow for additional features that kdb has lacked. Wessel described one possibility: 

A 2010 example of where kdb can be useful over kgdb is where you have a small netbook, no serial ports etc... and you are running X and your file system driver crashes the kernel. With kdb plus kms you can get an opportunity to see the crash which would have otherwise been lost from /var/log/messages because the crash was in the file system driver. 

While kgdb allows access to all of the standard debugging commands that gdb provides, kdb has a much more limited command set. One can examine and change memory locations or registers, set breakpoints, and get a backtrace of the stack, but those commands typically require using addresses, rather than symbolic names. Currently, the best reference for kdb commands comes from a [developerWorks article](<http://www.ibm.com/developerworks/linux/library/l-kdbug/>), though Wessel plans to change that. There is some [documentation](<http://kernel.org/pub/linux/kernel/people/jwessel/kdb/>) that comes with the patches, but a command reference will depend on exactly which pieces, if any, actually land in the mainline. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

It should be noted that one of the capabilities that was removed from kdb as part of the merger is the disassembler. It was x86 specific, and the new code is ""99% platform independent"", according to the [FAQ](<http://kgdb.wiki.kernel.org/index.php/KDB_FAQ>) about the merged code. Because kgdb is implemented for many architectures, rewriting it atop kdb led to support for many more architectures for kdb. Instead of just the x86 family, kdb now supports arm, blackfin, mips, sh, powerpc, and sparc. 

In addition, kgdb and kdb can work together. From a running kgdb session, one can use the gdb `monitor` command to access kdb commands. There are several that might be helpful like `ps` for a process list or `dmesg` to see log output. 

The FAQ [lists](<http://kgdb.wiki.kernel.org/index.php/KDB_FAQ#Why_merge_kdb_and_kgdb.3F>) a number of other advantages that would come from the merge, beyond just getting kdb into the mainline so that its users no longer have to patch their kernels, The basic idea behind the advantages listed is to unite the users and developers of kgdb and kdb so that they are all pulling in the same direction, because ""both kdb and kgdb have similar needs in terms of how they integrate into the kernel"". There have been arguments in the past about which of the two solutions is best, but, since they serve different use cases, having both available would have another benefit: ""No longer will people have to debate which is better, kdb or kgdb, why do we have only one... Just go use the best tool for the job."" 

Wessel notes that Ubuntu has enabled kgdb in recent kernels, which is something he would like to see done by other distributions. If kdb is available, that too could be enabled, which would make it easier for users to access the functionality: 

My other hope is that the new kdb is much easier to use in the sense that the barrier of entry is much lower. For example, someone with a laptop running a kernel with a kdb enabled kernel can use it as easily as: 

```
echo kms,kbd > /sys/module/kgdboc/parameters/kgdboc
        echo g > /proc/sysrq-trigger
        dmesg
        bt
        go
```

And voila you just ran the kernel debugger. 

In the example above, Wessel shows how to enable kdb (for keyboard (kbd) and KMS operation), then trap into it using sysrq-g (once enabled, kdb will also be invoked if there is a panic or oops). The following three commands are kdb commands for looking at log output, getting a stack backtrace, and continuing execution. 

The patches themselves are broken up into three separate patchsets: the first and largest adds the kdb infrastructure into `kernel/debug/` and moves `kgdb.c` into that directory, the [second](<http://lwn.net/Articles/374204/>) adds KMS support for kdb along with an experimental patch to do atomic modesetting for the i915 graphics driver, and the [third](<http://lwn.net/Articles/374205/>) allows kernel debugging via kdb or kgdb early in the boot process; starting from the point where `earlyprintk()` is available. Wessel is targeting 2.6.34 and, at least so far, the patches have been well received. The most recent posting is version 3 of the patchset, with a long list of changes made in response to earlier comments. Furthermore, an [RFC](<http://lwn.net/Articles/332611/>) about the idea last May gained a fair number of comments that clearly indicated there was interest in kdb and merging it with the kgdb code. 

Sharp-eyed readers will note some similarities between this proposal and the recent [utrace push](<http://lwn.net/Articles/371210/>). In both cases, an existing debugging facility was rewritten using a new core, but there are differences as well. Unlike utrace, the kdb/kgdb patches directly provide some lacking user-space functionality. Whether that is enough to overcome Torvalds's semi-hostile attitude towards kernel debuggers--though the inclusion of kgdb would seem to indicate some amount of softening--remains to be seen.
