---
title: Profiling kernel code coverage
url: https://lwn.net/Articles/284886/
date: "June 4, 2008"
category: Profiling
author: "By Jake Edge June 4, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
June 4, 2008

Measuring which lines of code get executed and how often can be a useful tool for debugging or testing. That capability has long been available for user space programs in the form of [gcov](<http://gcc.gnu.org/onlinedocs/gcc/Gcov.html>). A recent [patch](<http://lwn.net/Articles/284824/>) seeks to allow kernel hackers access to the same tool. 

There are three main components to making gcov work with the kernel: changing the build to add the `-fprofile-arcs -ftest-coverage` gcc flags, hooking up the gcc-generated code to record the coverage information, and providing a way for the kernel to output the data to user space. The `GCOV_PROFILE` kconfig option governs whether to include gcov into the build, while `GCOV_PROFILE_ALL` activates profiling for the entire kernel. If desired, individual directories and files can be selectively included or excluded from being instrumented. 

The new `kernel/gcov` directory contains the necessary functions to support the gcc-generated profiling code. This includes handling statically linked kernel code as well as kernel modules that are loaded. Information gathered from code in modules can be either preserved or discarded when they are unloaded. This will allow analysis of the module unloading path that could be useful for detecting resource leaks or other problems in that process. 

A user space program compiled for gcov will write a binary file to the filesystem for each source file that contains the data corresponding to the execution path through that file. The kernel needs to do that differently, so instead it writes to a file in [debugfs](<http://lwn.net/Articles/115405/>). Each source file that is compiled for gcov will store its information in `/sys/kernel/debug/gcov/path/file.gcda`, where `/sys/kernel/debug` is the debugfs mount point and `path` is the path to the file in the kernel tree. The individual `.gcda` files can also be written to, which will result in setting the accumulated data for that source file back to zero. 

Once the data has been gathered, gcov can be invoked to produce a file that annotates the source showing each line with the number of times it has been executed. [LCOV](<http://ltp.sourceforge.net/coverage/lcov.php>) is a graphical tool that can also be used to examine the coverage information. LCOV and the gcov kernel patches both come from the [Linux Test Project](<http://ltp.sourceforge.net/>) which has an extensive kernel test suite and is using gcov to expand the coverage of their tests. 

As part of the patch set, the [seq_file](<http://lwn.net/Articles/22355/>) interface has been extended to allow writing of arbitrary binary data to a virtual file. Currently, the seq_file interface is somewhat character oriented, so a function has been added to `fs/seq_file.c` to provide that ability: 

```
int seq_write(struct seq_file *seq, const void *data, size_t len)
```

As the prototype implies, it writes `len` bytes from `data` to the seq_file `seq`. 

Efforts to get gcov support into the kernel have [been around since 2002](<http://lwn.net/2002/0207/a/gcov-kernel.php3>), but the code was recently rewritten to be a better fit for recent kernels. In the patch, Peter Oberparleiter says ""due to regular requests, I rewrote the gcov-kernel patch from scratch so that it would (hopefully) be fit for inclusion into the upstream kernel."" One of the bigger changes is to move the user space interface for gcov from `/proc` into debugfs. 

It seems that the technical issues have largely been addressed in the third version of the gcov patch. It can provide useful information, especially for increasing the reach of test coverage--something that can only help reduce kernel bugs--so it could make for a nice kernel addition. Whether it will be picked up into linux-next or -mm and pushed towards an eventual mainline merge remains to be seen.
