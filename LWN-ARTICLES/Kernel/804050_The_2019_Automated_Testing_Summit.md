---
title: The 2019 Automated Testing Summit
url: https://lwn.net/Articles/804050/
date: "November 13, 2019"
category: "Development tools-Testing"
author: "By Jake Edge November 13, 2019 ATS"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
November 13, 2019

* * *

[ATS](<https://lwn.net/Archives/ConferenceByYear/#2019-Automated_Testing_Summit>)

This year saw the second edition of the [Automated Testing Summit](<https://events19.linuxfoundation.org/events/ats-2019/>) (ATS) and the first that was open to all. [Last year's ATS](<https://elinux.org/Automated_Testing_Summit_2018>) was an invitation-only gathering of around 35 developers (that was described in an [LWN article](<https://lwn.net/Articles/771782/>)), while this year's event attracted around 50 attendees; both were held in conjunction with the Embedded Linux Conference Europe (ELCE), in Edinburgh, Scotland for 2018 and in Lyon, France this year. The basic problem has not changed—more collaboration is needed between the different kernel testing systems—but the starting points have been identified and work is progressing, albeit slowly. Part of the problem, of course, is that all of these testing efforts have their own constituencies and customers, who must be kept up and running, even while any of this collaborative development is going on. 

#### Setting the stage

[ ![\[Tim Bird\]](https://static.lwn.net/images/2019/ats-bird-sm.jpg) ](<https://lwn.net/Articles/804501/>)

As with the first ATS, this edition was organized by Tim Bird and Kevin Hilman. Bird welcomed everyone to the conference then turned things over to Hilman for something of an overview of the "kernel testing landscape". Hilman started by noting that there were some gatherings and discussions at the [Linux Plumbers Conference](<https://www.linuxplumbersconf.org/>) (LPC) in September, which he [described in an email](<https://lists.yoctoproject.org/pipermail/automated-testing/2019-September/000499.html>) to the [automated-testing mailing list](<https://lists.yoctoproject.org/listinfo/automated-testing>). There were some themes that came out of those discussions, he said, which led to the title of his talk ([slides [PDF]](<https://elinux.org/images/3/36/ATS_2019.pdf>)): ""The bugs are too fast (and why we can't catch them)"". 

He gave a brief summary of the new kernel unit-testing frameworks that were discussed at LPC in order to bring attendees up to date on what kernel developers have been up to. The existing kernel test efforts, including [kselftest](<https://www.kernel.org/doc/html/latest/dev-tools/kselftest.html>), [Linux Test Project](<https://linux-test-project.github.io/>) (LTP), [syzbot](<https://github.com/google/syzkaller/blob/master/docs/syzbot.md>), and others, are likely pretty familiar to attendees, he said. The [KUnit](<https://kunit.dev/third_party/kernel/docs/>) framework ([LWN article from March](<https://lwn.net/Articles/780985/>)) has been merged into linux-next; it is a fast way to test kernel functionality in an architecture-independent way and can be run in user space with [user-mode Linux](<https://kunit.dev/third_party/kernel/docs/>) (UML). The [Kernel Test Framework](<https://github.com/oracle/ktf>) (KTF) is another unit-test framework that has been posted for comments. Since KUnit is headed for the mainline, though, the KTF project will need to figure out how to add its functionality to KUnit, Hilman said, since there won't be [multiple unit-test frameworks in the mainline](<https://lwn.net/Articles/790235/>). 

He then turned to the various testing initiatives that are currently active. The Intel [0-Day test service](<https://01.org/lkp/documentation/0-day-test-service>) is probably the longest running; it is "mostly Intel focused". The Linaro [Linux kernel functional testing](<https://lkft.linaro.org/>) (LKFT) has "quite a bit of in-depth testing but on a narrower set of hardware". The Red Hat [continuous kernel integration](<https://cki-project.org/>) (CKI) project has been around for a while, but has only recently been seen more publicly, he said; it is focused on testing stable kernels. And, of course, there is [KernelCI](<https://kernelci.org/>) that he cofounded; it was officially [announced as a Linux Foundation project](<https://lwn.net/Articles/803262/>) earlier in the week. 

[ ![\[Kevin Hilman\]](https://static.lwn.net/images/2019/ats-hilman-sm.jpg) ](<https://lwn.net/Articles/804502/>)

There is lots of testing going on, Hilman said, but there are a number of problems with that. One is that everyone is doing this testing off in their own corner; there is little collaboration between the efforts, which is the reason for the existence of ATS. All of the different players are testing areas that they care about, vendors are testing their hardware or software, developers test the platforms they care about, and so on, but the testing coverage of the kernel is concentrated; much of the kernel is tested, but everyone is generally testing the same parts of the kernel over and over. Broadening that coverage is an area that needs work. 

But even with the fragmentation, these test efforts are finding "lots and lots of bugs"; "can we actually keep up and fix the bugs as we find them?" He referred to an [LPC talk](<https://linuxplumbersconf.org/event/4/contributions/554/>) from Dmitry Vyukov (who was also present at ATS) that outlined some of the parameters of the size of the bug problem. Hilman used the statistics from that talk to note that more than 10% of the patches going into the kernel over the last few years carried a "Fixes:" tag, which means they are fixing a known bug that is identified in the tag. Not all patches that fix bugs use that tag, however, so the percentage of actual fixes is higher. 

Beyond that, the raw numbers of bugs being found are mind-boggling. In two years, syzbot has found 5,800 crashes by fuzzing the kernel; in doing so, it has only exercised around 7% of the kernel. Does the kernel community actually have the capacity to handle "all" of the bugs that could be found? Vyukov estimated that there are 20,000 new bugs added in each major kernel release. 

The LPC talk led to [discussion at the Maintainers Summit](<https://lwn.net/Articles/799134/>), which was held a few days later. Every kernel developer has their own workflow to handle and track patches, which does not scale when the number of bugs being reported grows rapidly. There are efforts underway to figure out ways to automate some of those of processes, Hilman said. The [workflows mailing list](<http://vger.kernel.org/vger-lists.html#workflows>) was created to discuss that; there are a lot of ideas on it, but "there is not a ton of consensus yet". 

Hilman said that the fragmentation in the testing landscape (and elsewhere) is one of the things that is preventing the community from digging out from under all of these bugs. He noted that the reason for ATS is to help fix that fragmentation, so that "we can actually collaborate on fixing the issues rather than dealing with understanding each others' frameworks [...] or competing with each others' frameworks". The ultimate goal is to stop as many bugs as possible from even entering the released kernels. Defragmenting testing is only addressing one half of the problem; the kernel workflow also needs to work to get there. 

An attendee noted that Hilman had talked a lot about kernel testing, but there is other testing being done as well. Bootloaders, user-space programs, and other components are being tested too; that kind of testing should continue. Hilman agreed; he focused on kernel testing mostly because the last few events that he was reporting on were also kernel-focused. There is definitely fragmentation in those other kinds of testing efforts, but "we gotta start somewhere" and kernel testing seemed like a good common denominator. 

#### Status updates

[ ![\[Milosz Wasilewski\]](https://static.lwn.net/images/2019/ats-wasilewski-sm.jpg) ](<https://lwn.net/Articles/804503/>)

A series of "lightning talks" was next on the agenda. These were short project overviews and updates for some of the different testing frameworks. There was more covered in each talk than is reported below; the portions that introduced the testing system were particularly helpful. 

Milosz Wasilewski was first, describing LKFT, which targets both the Arm and x86 architectures. It focuses on testing the stable kernels; LKFT was originally set up to help Greg Kroah-Hartman maintain the long-term stable (LTS) kernels, so it tests the LTS branches and the most recent stable branch as well. It has also expanded into testing the mainline and linux-next kernels over time. 

LKFT runs a variety of test suites, including LTP, kselftests, tests for [libhugetlbfs](<https://github.com/libhugetlbfs/libhugetlbfs>), and various performance tests, he said. That adds up to around 25,000 tests that get run for every release and around one million tests that get run every every week. In addition, there has recently been some testing of Android kernels, though that is somewhat "less advertised". 

Bird talked about the [Fuego test system](<http://www.fuegotest.org/>), which is targeted at testing on embedded devices. Fuego has its own Linux "distribution" that is based on Debian, with the [Jenkins](<https://jenkins.io/>) automation server, a test execution core, and a bunch tests installed on it. That is all wrapped up inside of a Docker container that runs on the host that is controlling the testing. 

The focus for Fuego is not on testing the upstream kernel, but is instead for doing "product testing"—"high-level integration testing and benchmarking". Various facilities needed for testing (e.g. the [netperf](<https://hewlettpackard.github.io/netperf/training/Netperf.html>) server) are integrated into the Fuego distribution, as is a cross-platform toolchain to build the tests from source. There are scripts to control test execution, including handling results parsing, analysis, and visualization. Multiple transports for talking with the device under test (DUT) are supported, including TCP, serial ports, and [Android Debug Bridge](<https://developer.android.com/studio/command-line/adb>) (adb). 

Fuego does not handle provisioning the device with the code to test; Bird hopes to be able to reuse the provisioning support from some other tool. His users mostly handle the provisioning as another Jenkins job in the test pipeline. The DUT is not required to have much in the way of capabilities in order to function with Fuego: just a shell with a limited set of commands, not including awk or sed, though grep is needed. All of those are available in [BusyBox](<https://busybox.net/>), he said. Some tests may require additional support on the DUT, however. 

Bird was followed by another familiar face as Hilman stepped up to talk about KernelCI. That project's goal is to test on a wide range of hardware platforms, he said, which makes it different from many of the other frameworks. Today, it is running tests on more than 250 boards, with systems-on-chip (SoCs) from more than 35 vendors, and for multiple architectures: x86_64, arm, arm64, mips, arc, and riscv. 

KernelCI tests multiple trees, including the mainline, linux-next, and the stable trees (as well as the stable release candidates). It also tests subsystem trees, maintainer and developer trees, the android-common tree, and the chrome-platform tree. It tests multiple kernel configurations, including all of the upstream defconfigs, and builds with multiple versions of both GCC and Clang. 

[ ![\[Veronika Kabatova\]](https://static.lwn.net/images/2019/ats-kabatova-sm.jpg) ](<https://lwn.net/Articles/804504/>)

For the most part, KernelCI is simply doing boot testing; it boots to a shell and does a few simple commands. For a subset of the boards, more functional tests are being added. The main thing is that the project does not want to develop and maintain its own test suites, but it is working with subsystems that have their own test suites to add them to KernelCI. 

Veronika Kabatova was up next to describe CKI, which has the goal of preventing bugs from getting into the kernel in the first place; finding existing bugs is nice, she said, but not the goal of CKI. Getting there is hard, however, so the project has started by testing trees in the upstream kernel. It is testing the trees of the newest stable branch, stable-next, arm-next, RDMA, and rt-devel; some testing of other trees is being done, like net-next and mainline, but the results are not being sent out. It runs tests for the x86_64, arm64, ppc64, ppc64le, and s390x architectures. Overall, CKI is meant to complement the other existing continuous integration (CI) solutions for the kernel. 

Unlike others, CKI does not use Jenkins; it uses the [GitLab CI](<https://docs.gitlab.com/ee/ci/>) system instead. For actually running the tests, [Beaker](<https://beaker-project.org/>) is used. Various kinds of input can trigger the system, including patches from [Patchwork](<http://jk.ozlabs.org/projects/patchwork/>), Git commits, [Koji](<https://fedoraproject.org/wiki/Koji>) builds, and more. Reports from CKI are sent to a mailing list. 

#### Kcidb

Several of the framework talks referred to [kcidb](<https://github.com/kernelci/kcidb>), which is something that came out of the [CKI hackfest](<https://cki-project.org/posts/plumbers-summary/>) that was held right after LPC. In his talk, Hilman mentioned that there is a huge corpus of test results that KernelCI has squirreled away but is never used. It would be nice to be able to do something with all of that data. Other projects have the same problem, of course. 

At the hackfest, KernelCI, LKFT, and CKI, got together to work on a simple JSON schema to describe test results. It is simply a starting point that can be used to gather all of this data into a common place, so that it can be processed with common tools to see what can be found within. There are tools to submit and query kcidb data in a [BigQuery](<https://cloud.google.com/bigquery/>) instance in the Google cloud. The "founding" three test frameworks are also working on adding a kcidb client to their reporting—as are other frameworks (Bird mentioned it for Fuego, at least). 

#### Plenty more

This article is only scratching the surface of the all-day conference, covering much of the first quarter of the day. ATS soon split into two tracks, as can be seen in the [schedule](<https://ats19.sched.com/>). There is a lot of information exchange going on at this point, trying to bring everyone up to speed on all of the other efforts, tools, frameworks, test hardware, and so on, which is reflected in the talk topics. But it seems abundantly clear that there is an enormous amount of defragmentation work to do. 

The current focus on kcidb was not even two month's old when ATS was held on October 31—because kcidb itself was brand new. There is some interest in working on common test definitions that can be shared among frameworks. Bird's vision is that some day there would be a "test store" that is akin to an app store, where users could browse through thousands of tests to choose the ones that fit their needs. But for now, that effort is on the back burner, simmering while the test-results piece is solidified—starting with kcidb. 

In the end, there is a lot of work to do and only a limited number of folks to do it—as is so often the case in the free-software world. In addition, it seems likely that LPC (and the CKI hackfest) being held so near in time to ATS split the participation to some extent. At the wrapup session at the end of the day, it was decided to try to focus on a single event, next year's LPC, rather than to reprise ATS for next year. 

During that wrapup session, the "key decisions" were gathered, which can be seen on the [ATS 2019 page](<https://elinux.org/Automated_Testing_Summit_2019>) of the Embedded Linux Wiki at elinux.org. In addition, Pengutronix folks took notes that they [posted](<https://lists.yoctoproject.org/pipermail/automated-testing/2019-November/000639.html>) to the automated-testing mailing list.

Over the years, kernel testing has most certainly gotten a _lot_ better, but there is still a long way to go. Collaboration seems like it will be pivotal in pushing kernel testing to the next level—and beyond. Gatherings like ATS, LPC, the CKI hackfest, and others will play an important role. 

[I would like to thank Linaro for travel assistance to attend the Automated Testing Summit in Lyon, France.]
