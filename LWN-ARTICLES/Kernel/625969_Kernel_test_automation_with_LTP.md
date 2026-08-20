---
title: Kernel test automation with LTP
url: https://lwn.net/Articles/625969/
date: "December 17, 2014"
category: "Development tools-Testing"
author: "December 17, 2014 This article was contributed by Cyril Hrubis"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

December 17, 2014

This article was contributed by Cyril Hrubis

Good test automation is a blessing that saves developers from repetitive tasks, reduces bugs introduced by human errors and, at the same time, decreases testing costs in the long term. [Linux test project](<http://linux-test-project.github.io/>) (LTP) is an established project that aims to bring test automation to Linux kernel development. 

In this article, I will briefly introduce LTP along with its history and structure. A second article will introduce the test library API. The motivation for writing them is to help kernel developers with the unpopular and sometimes neglected task of software testing. Increasing test coverage improves the development process, reducing the development effort and making software updates more predictable. This keeps developers happy by making more time available for the development of new interesting technologies and features. 

#### A bit of history and the current state

LTP was started in 2000 as a [joint open-source project by IBM, SGI, and OSDL](<https://web.archive.org/web/20010712183524/http://ltp.sourceforge.net/>) and was later joined by other interested parties. In 2001 it contained about 100 simple system call tests and a few test suites collected from other sources. As of today, it's maintained by SUSE, Red Hat, Fujitsu, and Oracle and gets contributions from a number of other companies and hobbyists. 

The goal of the project has always been ""to validate the reliability, robustness, and stability of Linux"". As that motto suggests, LTP focuses on functionality, regression, and stress testing for the Linux kernel and related features. Neither running benchmarks nor analyzing benchmark results are supported and there is no plan to add that support to LTP. Readers interested in benchmarks are advised to look into [MMTests](<https://github.com/gormanm/mmtests>) developed by Mel Gorman. 

A big problem for LTP is that the project goal is a bit too broad. There are two subproblems to that. The first is that LTP is relatively large (roughly 4000 C source files and around 500 shell scripts). Due to the size of the project, the content has historically varied in quality and quantity. Developers had complained about the unreliability of some of the tests. In recent years, significant effort has been spent on cleaning up that heritage, which dated back to the days of Unix wars. This was a reflection of the fact that IBM and SGI ported some of the code that became LTP from their commercial Unixes and released it under the GPL. Developers who tried LTP in the past and were unhappy with the experience are strongly encouraged to download a recent version and reevaluate. 

The second problem is completeness. LTP covers fair number of system calls, ioctls, sysfs, procfs interfaces, etc. but, given that the only documentation for some kernel interfaces is pieces of source code scattered around various subsystems, even estimating the coverage is a difficult task. Unfortunately, even the documentation we have is sometimes [incomplete](<http://git.kernel.org/cgit/docs/man-pages/man-pages.git/commit/?id=19402081af473eb70d21fbb8b89e1533c6456511>), [misleading](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=6900807c6b95dcb004902302b8ac5dbfbf6feb89>), or [wrong](<http://git.kernel.org/cgit/docs/man-pages/man-pages.git/commit/?id=a2cf88983114837a0673a00f76d12232d1606e07>). 

To give at least some impression of the coverage, which is quite possibly misleading, we can look at the overall number of test cases. The [latest stable tarball](<http://sourceforge.net/projects/ltp/files/LTP%20Source/ltp-20140828/ltp-full-20140828.tar.bz2/download>), released in August, contains 1047 system call test cases, 1605 POSIX conformance tests in a well-maintained fork of the [Open Posix Test Suite](<http://posixtest.sourceforge.net/>), a realtime test suite, various I/O stress tests (roughly 400), and network-related test cases, along with nearly a hundred test cases covering control groups (cgroups), various cgroup controllers, and namespaces. 

#### LTP design goals

LTP is designed to be dead simple; the primary design goals are: 

  * Each test is an executable.
  * Each test is as self-contained as possible.
  * Each test covers a well-defined assertion or a small group of similar assertions.
  * Each test runs automatically. (There is no need for manual setup nor input during the test run.)
  * Overall test status is passed as an exit value.
  * Additional information is printed to stdout.
  * Global parameters are passed via environment variables.

From the technical standpoint the languages of choice are C and portable shell. LTP adopted the [Linux kernel coding style](<https://www.kernel.org/doc/Documentation/CodingStyle>) and the development process centers around patch review on the [mailing list](<https://lists.sourceforge.net/lists/listinfo/ltp-list>). 

#### Getting and installing LTP

All released tarballs are stored on [SourceForge](<http://sourceforge.net/projects/ltp/files/LTP%20Source/>). These are time-based releases, made four times per year. Before the release of a tarball, the main repository is frozen for anything other than fixes for a week or two while the latest code is tested on several distributions. 

Then there is a Git repository on [GitHub](<https://github.com/linux-test-project/ltp>) that is updated nearly daily and, depending on how far it is from the previous release, it contains a few tens of new test cases and hundreds of fixes. Therefore, the latest Git is more suitable for testing upstream kernels than is the released tarball that may be a few months old. The Git code may be broken sometimes though, especially on older, but still maintained, distributions when the kernel is missing some of the functionality that the newly introduced test cases are testing. 

The installation process is pretty straightforward. The build configuration is done with an autotools `configure` script and the build is managed with `make`. LTP, by default, installs its files into `/opt/ltp/` where you will also find scripts to run the tests. 

To compile and install LTP from a Git snapshot you should do: 

```
$ git clone https://github.com/linux-test-project/ltp.git
        $ cd ltp
        $ make autotools
        $ ./configure
        $ make -j$(getconf _NPROCESSORS_ONLN)
        $ sudo make install
```

#### How to run LTP

Single test cases can be executed directly just by executing the binaries. A few of them will need `$CWD` in `$PATH` or additional parameters. If you are looking for a documentation on a particular test case, the best place to look is in the comment at the start of the test case source code. 

To run a set of test cases and to get a log file of the output you will need to use a test driver. By default, the test driver runs the default test [scenario](<https://github.com/linux-test-project/ltp/tree/master/scenario_groups/default>), which is a set of [runtest files](<https://github.com/linux-test-project/ltp/tree/master/runtest>) to execute. All runtest files that are part of the default scenario should contain reasonably stable tests. You will likely want to run only a particular subset of the runtest file(s) depending on the focus of the testing. 

Although the test driver works well, there is still a room for improvement. For example, integrating the Open POSIX Test Suite that, at the moment, can only be executed separately. 

The main run script is installed by default at `/opt/ltp/runltp`. This script is a wrapper around the `ltp-pan` test driver that runs test cases according to the runtest files. The `runltp` scripts has many optional parameters. Those used frequently include `-f filename` that selects only a single runtest file, `-s regexp` that runs only test cases whose name fits the regular expression, `-d /tmpdir` that selects a temporary directory for the test cases, and `-g filename.html` that causes `runltp` to produce HTML output into the file name given. 

The script is expected to be executed as root and the execution times range from minutes to hours, depending on a set of tests to be executed and the speed of the system under test. After the test run, the results can be located under the `result` directory; there is also a list of failed test cases located under the `output` directory. 

Historically, LTP contained test cases that were expected to fail. This is no longer true, with the exception of [three](<https://sourceware.org/bugzilla/show_bug.cgi?id=13701>) `pthread_rwlock` Open POSIX test cases. If any other LTP test case fails, it's either bug in the system or in the test—in either case it needs to be reported and fixed. 

#### Who uses LTP

Here at SUSE we mostly use the latest stable tarball as a part of the enterprise kernel validation for releases as well as for maintenance updates. Most of the time, that finds subtle changes in the interface between kernel and user space that either ends up as a test case fix or as a legitimate kernel bug. From my experience, kernel bugs are less common; although, as more and more test cases are fixed, the percentage of legitimate bugs grows as well. 

To find out about the rest of the Linux world, I started a [survey](<http://sourceforge.net/p/ltp/mailman/message/33021488/>) on the LTP mailing list which was quite successful—see for yourself the responses I got. Several indicated that LTP was used as part of automated testing of kernels, especially on non-x86 architectures. Others use subsets of the tests as a kind of smoke test that are typically limited to a short run duration (a few hours at most). In addition, LTP has been [mentioned](<https://lwn.net/Articles/597351/>) as one of the tools used when Linux was ported to the K1 architecture. 

Another use of LTP worth of mentioning is its [recent integration](<http://sourceforge.net/p/ltp/mailman/message/33021541/>) with the LKP+ project (also known as the [0-day kernel testing infrastructure](<https://lwn.net/Articles/514278/>)). That testing framework can catch bugs and determine which kernel commits are responsible, even before the commits reach a kernel release. Beyond that, here are a few upstream kernel commits that mentioned LTP in their Git commit log for 2014: 

  * [MIPS TLB copy&paste error](<https://github.com/torvalds/linux/commit/8393c524a25609a30129e4a8975cf3b91f6c16a5>)
  * [0-size extended attributes](<https://github.com/torvalds/linux/commit/4e66d445d0421a159135572a0ba44b75c7c4adfa>)
  * [ext4 zeroing of page during writeback](<https://github.com/torvalds/linux/commit/eeece469dedadf3918bad50ad80f4616a0064e90>)
  * [mm/readahead.c: fix do_readahead() for no readpage(s)](<https://github.com/torvalds/linux/commit/58d5640ebdb273cc817b0d0cda7bcf2efbbc2ff7>)
  * [NFS: fix the handling of NFS_INO_INVALID_DATA flag in nfs_revalidate_mapping](<https://github.com/torvalds/linux/commit/d529ef83c355f97027ff85298a9709fe06216a66>)
  * [sched: Fix up scheduler syscall LTP fails](<https://github.com/torvalds/linux/commit/39fd8fd22b3224ec6819d33b3e34ae4da6a35f05>)
  * [Revert a __copy_{to,from}_user_inatomic change](<https://github.com/torvalds/linux/commit/d11739e6d83dc17a6b54cfa23f8d7872d9ef82e2>) 

#### Conclusion

Although it wasn't easy, LTP has came a long way to what we have today and, as you can see, it already has been a useful tool for testing. Hopefully this article explained where we were and where we are today. That should get you started on running the tests; the next article will introduce the test library and will help with writing test cases.
