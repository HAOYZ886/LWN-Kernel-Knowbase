---
title: Topics in filesystem testing
url: https://lwn.net/Articles/1082342/
date: "July 15, 2026"
category: "Filesystems-Testing"
author: "By Jake Edge July 15, 2026 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
July 15, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

It should come as no surprise that a gathering of filesystem developers would discuss filesystem testing; it has been a mainstay of the [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) over the years and the 2026 summit was no exception. Ted Ts'o led the discussion this time; he had a few different topics to raise, including his perception of increasing regressions for ext4 in the stable kernels and what can be done to help reduce them. As [with](<https://lwn.net/Articles/789225/>) [other](<https://lwn.net/Articles/896523/>) [similar](<https://lwn.net/Articles/937830/>) [sessions](<https://lwn.net/Articles/982099/>) at the summit over the years, there is a lot of interest in collaborating on test inputs and outputs, but finding a way to centralize that information has so far eluded the filesystem community. 

Ts'o began by noting that he has been noticing more ext4 regressions in the stable kernels of late. Part of the reason is that the ext4 developers have been working on features like support for folios; some of those patches ""have subtle dependency requirements that aren't necessarily getting picked up by the automation"". 

[ ![\[Ted Ts'o\]](https://static.lwn.net/images/2026/lsfmb-tso-sm.png) ](<https://lwn.net/Articles/1082682/>)

Another factor is that patches are being backported into older kernels more frequently, possibly with the assistance of LLMs, he said. So he has seen features that were backported into the 6.1 and 6.6 stable kernels, which led to bugs in those kernels. Some of the bugs caused the kernel to crash on certain tests in the [fstests suite](<https://github.com/kdave/xfstests>). Since there were more than a dozen patches backported, it was ""quite painful to actually find those issues"". He wondered if other filesystems that had not opted out of the automated patch selection for stable kernels were also encountering that problems. 

He has set up a test runner that monitors the patches bound for stable kernels; it will run fstests on kernels with those patches. He has not had time to review the results and compare them to a baseline to find regressions, however. It is something he can automate, but has not gotten there yet. 

Ts'o said that any filesystems that can be tested with fstests in his test runner, ""which is most of them"", could be added into the mix for testing with the stable-kernel patch candidates. He has the capacity to do so and can provide reports via email so that more filesystems can be tested with the stable backports. He also put out the word that he was looking for a Python programmer to develop a program to compare the test output from two runs to find regressions between them. 

Ts'o has also been spending some time on automation for his [xfstests-bld](<https://github.com/tytso/xfstests-bld#xfstests-bld>) test appliance. He has added support for doing Git bisection, including for situations where the kernel crashes while running the tests. He would be happy to help get that set up for anyone interested. [Kdevops](<https://github.com/linux-kdevops/kdevops#table-of-contents>) is another option for filesystem developers to use. He suspects that there will be a lot more activity due to bug reports and patches from LLMs; ""testing is the only way we can stay on top of it"". 

He then opened the floor for others to share their ideas about filesystem testing. One attendee suggested a shared database of the test matrix and the test results, noting that the idea has come up before. Others agreed, but noted that the environments used for testing—real hardware, virtual machines, different kinds of storage, and so on—make it hard to compare results from testing efforts. 

Chuck Lever said that the kdevops project has an [archive of results](<https://github.com/linux-kdevops/kdevops#kdevops-tests-results>) from running fstests, which might be a good starting point. He had also just found out that the kernel networking subsystem (netdev) stores its continuous-integration (CI) test results in [patchwork](<https://patchwork.kernel.org/>), which has the ability to store data with the patches being tracked. Netdev is using that for storing its CI results. (More information can be found on the [Netdev Infrastructure for Patch Automation wiki](<https://github.com/linux-netdev/nipa/wiki/>).) 

Ts'o said that he had asked Konstantin Ryabitsev about setting up a mailing list for test results that would be archived at [lore.kernel.org](<https://lore.kernel.org/>). When the request was made a year or so ago, Ryabitsev was not enthusiastic about having automated test results stored that way, probably because of the volume of data that might be produced. If others thought there might be value in a list like that, Ts'o said that he could raise the idea again. 

Two attendees described dashboards that are used for testing reports in their companies. Ts'o suggested that any open-source efforts of that nature should be posted to the [fstests mailing list](<https://lore.kernel.org/fstests/>), since there may be other developers who would use them. Developing a central database for test results with a dashboard that can be used to monitor them is an idea that comes up at every summit, Lever said. He thought it might be a "moonshot", but perhaps the Linux Foundation could be enlisted to help make that happen. Ts'o thinks the foundation believes it is already solving the problem with the [KernelCI project](<https://kernelci.org/>), but that effort is not well-suited to filesystem testing. 

Ts'o said it might be easier to get some one-time funding to simply develop a tool, rather than creating a project like KernelCI, but for filesystem testing, which requires ongoing fundraising to maintain. He suggested that getting filesystem developers to agree on what is needed, maybe around a prototype that someone has vibe-coded, might lead to funding to create a production-ready version of the tool. ""We should put our heads together offline."" 

Another item that Ts'o wanted to raise was the files of test failures that he is maintaining. They are like fstests expunge files, which list tests that should not be run, but are based on the kernel version where the test does not pass. They cover per-filesystem-type failures as well as failures based on a combination of filesystem-type and test scenarios; they document tests that do not work in various long-term-stable (LTS) kernel versions and that likely never will work in those versions. 

He said that distributions tend to simply pick up an older version of fstests that corresponds with the kernel they are using. But he does not want to maintain multiple versions of fstests and thinks there is value in running the latest tests; the newer versions of fstests will sometimes point out which kernel version fixed a bug, which may indicate useful backports. Running a newer fstests version does lead to more noise in the results because there are more tests that will never pass on, say, 6.1 or 6.6. That is why he maintains the test-failure files. 

Those files currently live with his test appliance, but he wondered if they should move elsewhere and be maintained collaboratively. His focus is on ext4, so that's well-covered by the test-failure files. Lever suggested that the test-failure information be added to the fstests repository, but noted that there may be pushback to fix the tests instead. Ts'o said that the fstests maintainer has made it clear that they are not interested in tracking ""what got fixed in what versions"". There is a certain amount of sense to that because different people are using the tests in different ways; Ts'o only tracks LTS kernels, while distributions will want to track their kernels, which may diverge from the upstream kernel versions. 

Christian Brauner raised the problem of "flaky" tests, those that only pass sometimes. Ts'o said he has an internal version of his harness that allows tests to be marked flaky; if they fail, they are run three more times and are only reported if all of those fail. He has meant to add that feature into the public version, since it is useful, but has not found the time. 

Various people have their own versions of the expunge files for different kernel versions, so it would be nice to put them all together, Ts'o said. Since fstests is not the right place, maybe the kernel would be, he suggested. With that, the discussion wound down and the session ended. 

[I would like to apologize for any errors here. The acoustics in the room were problematic for both hearing and recording. Misunderstanding and misidentification may have resulted.]
