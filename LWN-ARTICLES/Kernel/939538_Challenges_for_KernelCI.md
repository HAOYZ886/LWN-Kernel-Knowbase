---
title: Challenges for KernelCI
url: https://lwn.net/Articles/939538/
date: "August 1, 2023"
category: "Development tools-Testing"
author: "By Jake Edge August 1, 2023 EOSS"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
August 1, 2023

* * *

[EOSS](<https://lwn.net/Archives/ConferenceIndex/#Embedded_Open_Source_Summit-2023>)

Kernel testing is a perennial topic at Linux-related conferences and the [KernelCI project](<https://kernelci.org/>) is one of the larger testing players. It does its own testing but also coordinates with various other testing systems and aggregates their results. At the 2023 [Embedded Open Source Summit](<https://events.linuxfoundation.org/embedded-open-source-summit/>) (EOSS), KernelCI developer Nikolai Kondrashov gave a presentation on the testing framework, its database, and how others can get involved in the project. He also had some thoughts on where KernelCI is falling short of its goals and potential, along with some ideas of ways to improve it. 

Kondrashov works for Red Hat on its [Continuous Kernel Integration](<https://cki-project.org/>) (CKI) project, which is an internal continuous-integration (CI) system for the kernel that is also targeting running tests for kernel maintainers who are interested in participating. CKI works with KernelCI by contributing data to its [KCIDB](<https://github.com/kernelci/kcidb>) database, which is the part of KernelCI that he works on. He noted that he was giving the talk from the perspective of someone developing a CI system and participating in KernelCI, rather than as a KernelCI maintainer or developer. His hobbies include embedded development, which is part of why he was speaking at EOSS, he said. 

[ ![\[Nikolai Kondrashov\]](https://static.lwn.net/images/2023/eoss-kondrashov-sm.png) ](<https://lwn.net/Articles/939977/>)

There are a number of different kernel-testing efforts going on, including the [Intel 0-day testing](<https://www.intel.com/content/www/us/en/developer/topic-technology/open/linux-kernel-performance/overview.html>), [syzbot](<https://github.com/google/syzkaller/blob/master/docs/syzbot.md>), [Linux Kernel Functional Testing](<https://lkft.linaro.org/>) (LKFT), CKI, KernelCI, and more. Each system has its own report email format and its own dashboard for viewing results. KernelCI has its own set of results from tests run in the labs it works with directly, while those results and the results from other testing efforts flow directly into KCIDB. Having a single report format and dashboard for all of the myriad of testing results is one of the things that the KCIDB project is working on. "Conceptually it is very simple", he said; various submitters send JSON results from their tests, the failures get reported by email to those who have subscribed to receive them, the results also get put into a database, which then can be displayed in the dashboard. 

Currently, KCIDB gets around 300K test results per day, from roughly 10K different builds. He briefly put up a screen shot of the [Grafana](<https://grafana.com/>)-based [KCIDB dashboard](<https://kcidb.kernelci.org/>) (which can be seen in his [slides](<https://static.sched.com/hosted_files/eoss2023/ef/Kernel%20CI%20%E2%80%93%20How%20Far%20Can%20It%20Go%20%E2%80%93%20EOSS%202023.pdf>) or the [YouTube video](<https://www.youtube.com/watch?v=KD0Hk6PghPw>) of the presentation). He also showed an example of the customized report email that developers and maintainers can get; the one he displayed aggregated the results from four different CI systems, with links to the dashboard for more information. 

#### CI metrics

He gave a somewhat simplified definition of CI for the purposes of his talk: test every change to a code base, or as many changes as possible, and provide feedback to the developers of that code. Given that, there are four metrics that can be defined for a CI system, starting with its coverage, which is how much functionality is being tested. The latency metric is based on how quickly the feedback is being generated, while the reliability measure is about "how much can you trust the results"—does a reported failure correspond to a real failure and does a "pass" really mean that? The last metric is accessibility, which is "how easy it is to understand the feedback"; that is, whether the reports provide enough information of the right kind to allow developers to easily track down a problem and fix it. 

Using those metrics, the ideal CI system would cover everything, provide instant feedback as changes are created, the feedback "is always true", and the report "just says what's broken, so you don't have to figure it out". On the flipside, the worst CI system "is not covering anything useful, takes forever, and never tells you the right thing, and you cannot understand what it is saying". That CI system "is worse than no CI", Kondrashov said. 

KernelCI obviously falls between those two extremes, so he wanted to look at the project in terms of those metrics. For coverage, "nobody really seems to know" how much of the kernel is being tested by the various systems that report to KernelCI. Each testing project has its own set of tests that it runs and there is no entity coordinating all of the testing. He has heard that some of the testing projects have done some measurements, but those results are not really available. He put up an [LCOV](<https://github.com/linux-test-project/lcov>) coverage report that was generated from the CKI tests for Arm64. It showed an overall coverage of 12%, but that was only for the most important subset of the kernel tree. 

In an unscientific sampling of the mailing lists, he sees latencies of several hours after a change is posted, "which is quite good", up to "a few weeks"; the latencies are typically faster for changes that are actually merged into a public branch. Pre-merge testing is rare overall. 

The results are not particularly reliable, however. Many people who run CI systems have to do manual reviews of the test results before sending them to developers, "because things go wrong quite often". The accessibility of the results "is quite good in places" as some CI systems make an effort for their results to be understandable and actionable. 

There are certain "hard limits" on what can actually be accomplished. In terms of coverage, the amount of hardware that is available to be used for testing is a hard limit; the kernel is an abstraction over hardware, so it needs to be tested on all sorts of different hardware. The latency of feedback is also limited by hardware availability; more hardware equates to more tests running in parallel, which reduces the time for producing feedback. 

The reliability of the tests is governed by the reliability of the hardware and of the kernel, "but tests contribute to improving kernel reliability, so that's good". The reliability of the tests themselves would also seem to be a big factor here, though Kondrashov did not mention it. The limits of accessibility are partially governed by hardware availability, yet again, because it is difficult to fix a bug that is reported on hardware that the developer does not have access to. The complexity of the kernel also plays a role in limiting the accessibility of the results. 

#### Challenges

He thinks that there are a lot of people who want to write tests, a lot of tests already in the wild, and a lot of companies that have test suites, all of which could lead to more coverage, but integrating those new tests is being held back by other problems. 

Doing CI on code that has not yet been merged is dangerous. Anybody can post to the mailing list, so picking up those patches to test can cause problems: "you don't want them to start mining Bitcoin and you don't want them to wreck your hardware". The need for "slow human reviews" of the results also contributes to the latency problems. 

He thinks that a big reason why the tests can be unreliable is because they get out of sync with the kernel being tested. Kernels change, as do the tests, but the lack of synchronization means that a test may not be looking for the proper kernel behavior. That leads to tests that repeatedly fail until the two get back in sync; meanwhile, the maintainers do not want to hear about the repeated failures that are not actually related to real bugs in their code. Nobody wants to "waste their time investigating a problem that they had nothing to do with". 

The main challenge he sees for accessibility is the proliferation of report formats and dashboards, which makes it difficult for developers. That is something that he thinks KCIDB can improve. 

The challenges also compound: low reliability and accessibility lead to low developer trust in what the CI system produces. If a developer knows that the tests often fail due to problems completely outside of their control, "their trust and interest for these results plummets". Likewise, if the reports are hard to understand because the developer does not have access to the hardware where it broke, or the reports leave out important information, they will be ignored. That means the results will not be used for gating patches into the kernel. Since the results are generally ignored, the test developers do not get feedback about the tests, so the tests do not improve, and any actual bugs that the tests do find are not acted upon; the whole improvement feedback loop breaks down. 

High latency also leads to a lack of gating; you cannot wait a week for test results to decide whether a patch is sensible to be merged. That leads to bugs getting into the kernel that would have been caught in a lower-latency system. That all leads to greater latency as time is wasted on finding and fixing bugs that could have been detected; the extra time spent cannot go into improving the tests. "It's a vicious loop", he said. 

He summarized his takeaway from the challenges with a [meme](<https://imgflip.com/memegenerator/Too-Damn-High>): "Feedback latency is too damn high!" After that, though, he wanted to move on to what can be done to fix the problems: "that's enough gloom". 

#### What to do?

First up was a look at what _cannot_ be done, however. The kernel community is not a single team, working for a single employer; that is also the case with most other open-source projects. It all means that open-source developers cannot be forced to look at test results. In a company, you can bootstrap the testing into the development process by getting the tests just good enough to start gating merges on them; after that, the tests start improving and the positive feedback loop initiates. "After a bit of fighting and stalling, it starts up." In an open-source project, though, the tests need to be in good shape in order to gain developer trust; "without developer trust, it's not going to work". 

Turning to things that _can_ be done, he started with coverage. Companies have the most hardware, so attracting more companies into the testing fold will lead to more hardware, more tests, and more results, thus more coverage. Companies that have their own CI system and want to contribute to KernelCI can send their results to KCIDB. Another way to contribute is by setting up a [LAVA](<https://validation.linaro.org/>) lab and connecting it with KernelCI; developers will be able to submit trees and tests to be run on the hardware. The right place to get started with either of those is the [KernelCI mailing list](<https://lore.kernel.org/kernelci/>). 

Kondrashov said that he thinks more pre-merge testing is needed to try to head off bugs before they get into public code and to shorten the feedback loop for developers. There are multiple approaches to doing pre-merge testing; some are using [Patchwork](<https://github.com/getpatchwork/patchwork>) to pick up patches from the mailing list for testing, which is working well. There is still a problem with authentication, however, since anyone can send a patch to the list; some patches could be malicious. 

There are around 50 entries in the kernel `MAINTAINERS` file that refer to a GitHub or GitLab repository. Those systems provide a way to authenticate the patches that are submitted and connect them to a CI system. Something that KernelCI is exploring is to add integration with those "Git forges" so that, for example, there could be a GitHub Action that submits a patch to KernelCI and gets back a success or failure indication. The benefit is that those patches can be tested on real hardware as part of the pre-merge workflow. 

If that all can be made to work, he would like to encourage more maintainers to use the forges. "I know this is controversial, it's been discussed to death in the community." But a few kernel trees are already using the pull-request-based workflow; he thinks more could benefit from doing so. The "selling point" is the CI integration and early testing of pull requests. 

In order to get the process going, "CI systems have to start talking to maintainers". A CI system can offer to test a staging branch from the maintainer's repository; the maintainer's merge of a patch into their branch provides the authentication. That is not pre-merge testing, but is a starting point to help prove that the CI system and its tests are reliable and useful. To start with, a few of the most stable tests can be chosen. The KCIDB subscription feature will allow developers to get reports of other, related test results; users can filter the reports that they get on a variety of criteria, such as Git branch, architecture, compiler, tester, and so on. 

There are so many tests and so many failures that manually reviewing all of them is inefficient. CI systems are starting to set up automatic triaging to analyze the results in order to more efficiently find real problems. KCIDB is working on such a system, but other CI efforts, such as the [Intel GFX CI](<https://intel-gfx-ci.01.org/>) (for Intel graphics), CKI, and syzbot, already have working versions for this triage. The best triaging is currently done by syzbot in order to not emit multiple reports of what are the same underlying bug by analyzing the kernel log output of the crash. 

Another controversial suggestion that he has is to avoid the synchronization problem between the kernel and its tests by moving more tests into the kernel tree. That allows fixes or changes to the kernel functionality to come with the corresponding changes to the tests. He suggested starting with popular, well-developed tests, such as those from the [Linux Test Project](<http://linux-test-project.github.io/>) (LTP). In order to make that work, though, it needs to be integrated into the kernel documentation and best practices, so that the tests become a "more official" part of the kernel workflow. 

Currently, LTP is being run on mainline, stable, and other kernels, so it has to be able to handle all of the test differences among them. If those tests got integrated into the kernel tree, that would no longer be needed; the tests in the tree would (or should) simply work for that branch. If a fix that gets backported to a stable branch changes a test somehow, the test change would be backported with it; that would greatly simplify the tests. In order to keep those in-tree tests functioning well, they would need to be prioritized in the CI systems, so that the feedback loop for the tests themselves is shortened as well. 

Accessibility can be improved by standardizing the report formats and dashboards. The KCIDB project is working on some of that, but needs feedback from maintainers and developers. He also encouraged people to get involved with the [development of KCIDB](<https://kernelci.org/docs/kcidb/developer_guide/>) to help make it better. 

In the Q&A after the talk, several attendees agreed with Kondrashov's analysis and suggestions. There were also invitations to work with other testing efforts, such as for the media subsystem. Finding a way to allow regular developers to test their code on a diverse set of hardware was also deemed important, but depends on being able to authenticate the requester, Kondrashov reiterated; the Git forges provide much or all of the functionality needed for that. He closed by noting that there are few who are working on KCIDB right now, largely just Kondrashov—who is busy with other Red Hat work—and an intern, so there is a real need for more contributors; he has lots of ideas and plans, but needs help to get there. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for assisting with my travel to Prague.]
