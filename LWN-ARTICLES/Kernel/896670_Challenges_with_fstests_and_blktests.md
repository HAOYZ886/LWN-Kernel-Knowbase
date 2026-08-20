---
title: Challenges with fstests and blktests
url: https://lwn.net/Articles/896670/
date: "June 1, 2022"
category: "Block layer-Testing; Filesystems-Testing"
author: "By Jake Edge June 1, 2022 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
June 1, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

The challenges of testing filesystems and the block layer were the topic of a combined storage and filesystem session led by Luis Chamberlain at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). His goal is to reduce the amount of time it takes to test new features in those areas, but one of the problems that he has encountered is a lack of determinism in the test results. It is sometimes hard to distinguish problems in the kernel code from problems in the tests themselves. 

He began with a request to always use the term "fstests" for the tests that have been known as "xfstests". The old name is confusing, especially for new kernel developers, because the test suite has long been used for testing more than just the XFS filesystem. It is not just new folks, though; even at previous LSFMMs, he has seen people get confused by the "xfs" in the name. 

[ ![\[Luis Chamberlain\]](https://static.lwn.net/images/2022/lsfmm-chamberlain-sm.png) ](<https://lwn.net/Articles/896745/>)

He noted that it takes ten years or more to stabilize a new filesystem; long ago he set an objective for himself to try to help with that problem. One of the ways to do so is to reduce the amount of time that it takes to run filesystem tests. Along the way, he decided to try to reduce the time it takes to test new features in the block layer as well. 

One of the differences he has observed for fstests and blktests versus tests for the rest of the kernel is that their determinism is lower. The KUnit tests are "extremely deterministic", while the kernel selftests are highly deterministic, though they do sometimes fail unexpectedly. On the other hand, fstests and blktests are just the opposite; they can be "extremely non-deterministic", he said. 

One of the takeaways from his findings is that the time spent on "testing" needs to be divided properly. There are four separate parts of that, which should all get roughly equal amounts of time: test design, tracking results, reporting bugs, and fixing low-hanging fruit. The kernel development community is "actually pretty good at test design", but does not really spend enough time on the other parts of the testing puzzle. 

He has worked on the [kdevops project](<https://github.com/mcgrof/kdevops>) to try to make some of that better. It uses Kconfig for its configuration and allows users to choose between cloud or local virtualization for bringing up systems for kernel testing. But that was not the topic of the session, he said, rather the topic is the lessons that he has learned from that effort. 

One example of non-deterministic behavior is an ext4 test that fails once in 300 runs of fstests. When he asks filesystem developers how many times they run fstests in a loop, he gets a funny look, he said; but if you do run it in a loop, you will find some of these sporadic failures. Another example was a failure in blktests one time out of 80 because of an RCU stall. It turned out to be a problem in the QEMU zoned-device emulation, but that false positive in blktests helped track down the problem. 

Another example was in the ~~"block/000"~~ "block/009" test in blktests, which would fail once out of 669 times. It took around eight months to track down the problem and reach a consensus on the fix. Jan Kara merged a fix for 5.12 that could potentially be backported to earlier stable kernels, but it would be difficult to do because the patches are complex. 

Another failure that turned up in both blktests and fstests somewhat randomly is an example of the low-hanging fruit, he said. The error came about because of a longstanding problem removing kernel modules; the test tried (and sometimes failed) to remove the `scsi_debug` module. The underlying bug will be fixed in kmod soon by adding a more patient module remover, but it points to another problem: fstests and blktests should not require modules to be unloaded so that unrelated problems do not introduce sporadic failures of this sort. 

But others in the room said that it was important to ensure that the cleanup was done correctly, for example with NVMe devices. There was some discussion of whether that kind of testing was truly useful and whether module unloading was the right way to go about it, but no real consensus emerged. Josef Bacik said that it was important to focus on "testing the thing that we care about" and not to let unrelated problems muddy the waters by way of side effects. 

There are also some problems with the error reporting in fstests, Chamberlain said. There are two kinds of files associated with each test, a `.bad` file and another in the Junit format, that do not always agree. So both types of files need to be processed in order to find the errors associated with a particular test. Blktests is better in that respect, he said, at least partly because it is a newer test suite. 

Ted Ts'o said that tests with errors in one type of file and not the other are simply test bugs that should be fixed; the test runners could perhaps be changed to process both, as well, but the tests should be updated to have the right information. There was also some discussion of saving dmesg output when there are test failures. Bacik said that fstests has an option to always save that output even if the test passes, which can be useful; Omar Sandoval said that if blktests did not have a similar option, it would be added. 

To try to investigate the failure rates of some of the tests, Chamberlain runs fstests and blktests in a loop for 100 iterations on each. For running fstests on all of the filesystems, that loop takes five or six days, while the loop takes a single day for blktests. Tests that do not pass for all of the test runs can be removed from the baseline while they are being investigated. 

Ts'o cautioned that there are different goals for running these test suites, however. A QA person who is "trying for the platonic ideal of zero bugs" may have to do multiple runs looking for bugs that only appear infrequently. But, from his company's perspective, it does not make sense to try to detect those kinds of bugs since he does not have the budget to hire enough people to track them all down. 

Instead, his testing focuses on running tests on the hardware that is being used in production to try to find the kinds of bugs that will occur in that scenario. So Ts'o said he has different goals than Chamberlain does, though the work that Chamberlain is doing is valuable. Ts'o said that he is trying to "maximize bang for the buck" to produce the highest-quality kernel he can afford given his budget. Chamberlain agreed that there is a need to prioritize the work based on the goals of the organizations involved, but as Ts'o noted, this kind of work requires lots of resources. 

Moving onto another subject, establishing a baseline for a new filesystem takes one or two months, Chamberlain said. Not having a public baseline for a filesystem should be seen as a technical debt within the community. But it takes time and resources to investigate the test failures, so dropping failing tests to establish a "lazy baseline" is needed. 

Another problem that he sees is that tests that _should_ fail for a given configuration or filesystem should be annotated so that they can be run and the failure verified. But others disagreed, saying that known failures should be turned into separate tests to demonstrate the correct behavior. Bacik worried that it would simply introduce further uncertainty into the tests. The session ran out of time, but Bacik scheduled another session later in the day to discuss other problem areas for testing.
