---
title: Testing and the stable tree
url: https://lwn.net/Articles/789225/
date: "May 28, 2019"
category: "Development model-Stable tree; Regression testing"
author: "By Jake Edge May 28, 2019 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 28, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The stable tree was the topic for a plenary session led by Sasha Levin at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). One of the main areas that needs attention is testing, according to Levin. He wanted to discuss how to do more and better testing as well as to address any concerns that attendees might have with regard to the stable tree. 

There are two main things that Levin is trying to address with the stable tree: that fewer regressions are released and that all of the fixes get out there for users. In order to pick up fixes not marked for stable, he is [using machine learning](<https://lwn.net/Articles/764647/>) to identify candidate patches for the stable trees. Those patches are reviewed manually by him, then put on the relevant mailing list for at least a week; if there are no objections, they will go into the stable tree, which is under review for another week, then they are released. 

There have been some concerns expressed that the stable kernel is growing too much, by adding too many patches, which makes it less stable. He strongly disagrees with that as there is no magic limit on the number of patches that, if exceeded, leads to an unstable kernel. It is more a matter of the kind of testing that is being done on the patches proposed for the stable kernels. 

[ ![\[Sasha Levin\]](https://static.lwn.net/images/2019/lsf-levin-sm.jpg) ](<https://lwn.net/Articles/789377/>)

Levin noted that Darrick Wong and Dave Chinner (neither of whom were present at LSFMM this year) have expressed worries about the kind of testing that takes place. He has been working with them on trying to improve the testing of stable patches for XFS and other filesystems. Luis Chamberlain (formerly Rodriguez) has been working on a new tool to run [xfstests](<https://github.com/kdave/xfstests>) on the stable patches with various kernel configurations called [oscheck](<https://gitlab.com/mcgrof/oscheck>); right now, it targets XFS, but Levin would like to see it expand to testing other filesystems and get integrated with the [KernelCI](<https://kernelci.org/>) project. 

He does not have a good solution for testing I/O (or storage) and memory management at this point, however. There is the [blktests](<https://github.com/osandov/blktests>) framework for storage testing, but he has not looked into it yet. There is nothing that he knows about for memory-management testing, however; he would be happy to hear suggestions. 

Michal Hocko said that he was not happy with the idea of "automagic" backports of memory-management patches for stable kernels. He said that the memory-management developers are quite careful to mark patches that they think should make their way into the stable trees; adding others is just adding risks. 

There is no set of tests to detect regressions, he said; problems will not be found until you run real workloads on top of those kernels. There have been several cases where dubious patches were picked up for stable, so he does not think automatic patch selection should be used. SUSE has found that the stable trees are bringing in less stability and he thinks that is because you need a human brain to evaluate each of the patches. 

Levin said that he agrees that the memory-management developers do a "great job" of marking patches for stable; there were only 26 other memory-management patches in the last year that were proposed for the stable tree. Users expect that things may break a little bit in a stable kernel, he said, but they are afraid of the big updates like moving to a new kernel series. That's why it takes months or years for users to upgrade to a new kernel; if the changes are relatively small, it is less scary for users. If kernel developers hide scary patches only in newer kernel series, users simply won't upgrade—there are still users on 2.6.32, after all. 

But if distributions aren't using the stable kernels, Rik van Riel asked, who is? Levin said that the enterprise distributions from Red Hat and SUSE do not use them, but that Canonical, Android, and others do use the stable kernels. 

Steve French said that there are multiple filesystems that regularly tested with xfstests, including ext4, Btrfs, and SMB/CIFS. From his perspective it is fine to include more patches into stable, but he wondered if there is a mechanism to inform the filesystem developers that a regression test for a set of patches needs to be run. The developers could trigger such a test run if the stable maintainers could point them at a branch, he said. 

Filesystem developers are comfortable with backporting large sets of patches when they are able to run their tests, but French said he does not know when the stable kernels come out. Levin said that he is happy to extend the timing of the release, if needed; there are also ways to trigger builds and tests in other systems based on a stable candidate. French said that it takes around seven hours to run the tests for SMB/CIFS; he is not sure how long it takes for ext4 and Btrfs, but suspects it is roughly the same. 

Ted Ts'o said that the automation piece is what is needed. Currently, he would need to see the stable release-candidate email, and then personally download and build the kernel. Once that has been done, running the tests is easy: he uses nine virtual machines (VMs) and it takes about two or three hours of wall-clock time. Then some human needs to look at the results. If it were automated or someone were paid to do that work, it would happen; automating as much as possible will help. 

Levin again referred to the oscheck work that Chamberlain is doing. Ext4 would make a nice addition to that, Levin said. They are both willing to customize the process to make it easy for additional filesystems to come on board. It is mostly a matter of gathering the right "expunge lists" (i.e. xfstests that should not be run) for ext4 and other filesystems. In addition, those lists evolve as bugs are fixed, features are added, and so on; Levin said that Chamberlain's tool has ways to handle that. 

Chris Mason wondered why the Intel [O-Day automated testing](<https://01.org/lkp/documentation/0-day-test-service>) was not being used and that something new was being created instead. Levin said that the code for the Intel effort is only semi-open, so he has been working with KernelCI, which is all open source; the goal is to integrate with that effort. Chamberlain said that the 0-Day bot does run xfstests, but expunge-list management is a problem area for it. 

Levin said that he thinks that resources for running tests is becoming less of an issue; KernelCI has money and resources for testing, for example. Automated testing is cheaper; humans are needed to review code, but finding human time for review is difficult in some subsystems. Memory management is one of those, so automated tests that could at least confirm that a stable candidate "basically works" would be useful. 

Mel Gorman said that it "will be tricky" to come up with such tests. A basic round of memory-management tests takes one or two days, while a middling set takes three or four days; the full set of tests (which still leaves some stuff out) takes two weeks or so. These tests require a lot of CPU time, though they are fully automated. Levin thought that three days for testing would be workable. 

The problem is that mistakes in the memory-management subsystem manifest as performance problems, so the tests measure performance in various ways, Gorman said. The results are "much more subtle" than a simple pass/fail as other test suites have. The full set of [MMTests](<https://github.com/gormanm/mmtests>) takes three weeks, Gorman said. It would be nice if there was a way to characterize whether the tree has regressed "too much", Levin said, so that someone could start looking at that. 

Moving back to the filesystem tests, Ts'o said that managing the expunge (or exclude) lists is going to be a major headache; it is not something that Chamberlain can do alone. Those files also need to be commented so that it is clear why tests are being skipped. Doing that kind of testing will be an ongoing effort that requires a lot of humans, Ts'o said. Chamberlain agreed, noting that he is currently handling XFS, but that other people are needed for other filesystems. 

The merits of an include list versus an exclude list were also discussed. Ts'o said that an include list will never get updated when people get busy, so new tests won't end up being run. With an exclude list, failures are noise that will get attention. French said that it is important to test with both kinds of lists, but it is equally important to collect and use different kernel configurations. 

The matrix of exclude and include lists, along with local kernel configurations, for each of the kernel series of interest is going to be large, Levin said. It is important to remember that most users are not running the latest kernels, so bugs that get fixed in 5.x kernels are not reaching users unless they are backported to older kernels, he said. No one is running the kernels released by Linus Torvalds other than perhaps on their laptops, but certainly not at scale. 

Experience from other test projects, such as the [Linux Test Project](<https://linux-test-project.github.io/>) (LTP), shows that not running tests has its faults as well, Amir Goldstein said. LTP annotates its tests with minimum kernel versions, but still runs them expecting failure. Chamberlain noted that he started by running all of the tests, then documenting which failed and why. But Ts'o said that may not be workable in all cases; there are tests on the exclude list for xfstests because they crash the kernel for certain configurations. Or they take an inordinate amount of time under certain configurations (e.g. 1KB block size); that is why the exclude-list entries should be documented. 

As time expired for the session, Levin said he was hoping to talk to any attendees who had thoughts about integrating tests they already use into KernelCI. If those tests get into the framework, the stable team can point it at candidate trees to hopefully get better testing—and detect any regressions—before the release.
