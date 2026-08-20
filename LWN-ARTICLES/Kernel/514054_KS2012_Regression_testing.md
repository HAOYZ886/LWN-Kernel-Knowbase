---
title: "KS2012: Regression testing"
url: https://lwn.net/Articles/514054/
date: "August 30, 2012"
category: "Development tools-Trinity; Regression testing; User-space API-Testing"
author: "By Michael Kerrisk August 30, 2012 2012 Kernel Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Michael Kerrisk**  
August 30, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

The "regression testing" slot on day 1 of the 2012 Kernel Summit consisted of presentations from Dave Jones and Mel Gorman. Dave's presentation described his new fuzz testing tool, while Mel's was concerned with some steps to improve benchmarking for detecting regressions.   

#### Trinity: intelligent fuzz testing

Dave Jones talked about a testing tool that he has been working on for the last 18 months. That tool, [Trinity](<http://codemonkey.org.uk/projects/trinity/>), is a type of system call [fuzz tester](<http://en.wikipedia.org/wiki/Fuzz_testing>). Dave noted that fuzz testing is nothing new, and that the Linux community has had fuzz testing projects for around a decade. The problem is that past fuzz testers take a fairly simplistic approach, passing random bit patterns in the system call arguments. This suffices to find the really simple bugs, for example, detecting that a numeric value passed to a file descriptor argument does not correspond to a valid open file descriptor. However, once these simple bugs are fixed, fuzz testers tend to simply encounter the error codes (`EINVAL`, `EBADF`, and so on) that system calls (correctly) return when they are given bad arguments. 

What distinguishes Trinity is the addition of some domain-specific intelligence. The tool includes annotations that describe the arguments expected by each system call. For example, if a system call expects a file descriptor argument, then rather than passing a random number, Trinity opens a range of different types of files, and passes the resulting descriptors to the system call. This allows fuzz testing to get past the simplest checks performed on system call arguments, and find deeper bugs. Annotations are available to indicate a range of argument types, including memory addresses, pathnames, PIDs, lengths, and so on. Using these annotations, Trinity can generate tests that are better targeted at the argument type (for example, the Trinity web site notes that powers of two plus or minus one are often effective for triggering bugs associated with "length" arguments). The resulting tests performed by Trinity are consequently more sophisticated than traditional fuzz testers, and find new types of errors in system calls. 

Ted Ts'o asked whether it's possible to bias the tests performed by Trinity in favor of particular kernel subsystems. In response, Dave noted that Trinity can be directed to open the file descriptors that it uses for testing off a particular filesystem (for example, an ext4 partition). 

Dave stated that Trinity is run regularly against the `linux-next` tree as well as against Linus's tree. He noted that Trinity has [found bugs](<http://codemonkey.org.uk/projects/trinity/bugs-found.php>) in the networking code, filesystem code, and many other parts of the kernel. One of the goals of his talk was simply to encourage other developers to start employing Trinity to test their subsystems and architectures. Trinity currently supports the x86, ia64, powerpc, and sparc architectures. 

#### Benchmarking for regressions

Mel Gorman's talk slot was mainly concerned with improving the discovery of [performance regressions](<https://lwn.net/Articles/509577/>). He noted that, [in the past](<https://lwn.net/Articles/249054/>), ""we talked about benchmarking for patches when they get merged. But there's been much inconsistency over time."" In particular, he called out the practice of writing commit changelog entries that simply give benchmark statistics from running a particular benchmarking tool as being nearly useless for detecting regressions. 

Mel would like to see more commit changelogs that provide enough information to perform reproducible benchmarks. Leading by example, Mel uses his own benchmarking framework, [MMTests](<http://www.csn.ul.ie/~mel/projects/mmtests/>), and he has [posted](<http://thread.gmane.org/gmane.linux.kernel/1315993/focus=81491>) historical results from kernels 2.6.32 through to 3.4. What he would like to see is changelog entries that, in addition to giving benchmark results, identify the benchmark framework they use and include (pointers to) the specific configuration used with the framework. (The configuration could be in the changelog, or if too large, it could be stored in some reasonably stable location such as the kernel Bugzilla.) 

H. Peter Anvin responded that ""I hope you know how hard it is for submitters to give us real numbers at all."" But this didn't deter Mel from reiterating his desire for sufficient information to reproduce benchmarking tests; he noted that many regressions take a long time to be discovered, which increases the importance of being able to reproduce past tests. 

Ted Ts'o observed that there seemed to be a need for a per-subsystem approach to benchmarking. He then asked whether individual subsystems would even be able come to consensus on what would be a reasonable set of metrics, and noted that those metrics should not take too long to run (since metrics that take a long time to execute are likely not to executed in practice). Mel offered that, if necessary, he would volunteer to help write configuration scripts for kernel subsystems. From there, discussion moved into a few other related topics, without reaching any firm resolutions. However, performance regressions are a subject of great concern to kernel developers, and the topic of reproducible benchmarking is one that will likely be revisited soon.
