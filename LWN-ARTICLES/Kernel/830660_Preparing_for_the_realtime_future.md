---
title: Preparing for the realtime future
url: https://lwn.net/Articles/830660/
date: "September 9, 2020"
category: "Development model-Stable tree; Development tools-Testing; Realtime"
author: "By Jake Edge September 9, 2020 LPC"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
September 9, 2020

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2020-Linux_Plumbers_Conference>)

Unlike many of the previous gatherings of the Linux realtime developers, their microconference at the virtual [2020 Linux Plumbers Conference](<https://linuxplumbersconf.org/2020/>) had a different feel about it. Instead of being about when and how to get the feature into the mainline, the microconference had two sessions that looked at what happens _after_ the realtime patches are upstream. That has not quite happened yet, but is likely for the 5.10 kernel, so the developers were looking to the future of the stable realtime trees and, relatedly, plans for continuous-integration (CI) testing for realtime kernels. 

#### Stable trees

Since the realtime patch set will be fully upstream "any release now", a plan needs to be made for what will happen with the realtime stable (RT-stable) kernels, Mark Brown said to start his session. Currently, there are RT-stable kernels maintained for each of the long-term support (LTS) stable kernel versions; the realtime patch set is backported to those kernels. But once the patches are in the mainline, there will be no longer be a realtime tree to backport. 

[ ![\[Mark Brown\]](https://static.lwn.net/images/2020/lpc-brown-sm.png) ](<https://lwn.net/Articles/831102/>)

He wondered if people should simply be told to use the mainline stable kernels if they want the realtime feature. If so, any realtime performance regressions that occur in the stable trees will need to be addressed and those fixes will need to be accepted by the stable maintainers. Realtime developers will need to help with any conflicts that arise in backporting fixes to the stable kernels as well. 

Testing is another area that will need to be handled; in particular, realtime performance needs to be tested as part of the stable release process. Right now, Greg Kroah-Hartman largely outsources testing of specific use cases and workloads on stable kernels to those who are interested in ensuring those things continue to function well. Testing of realtime performance will need to be part of that. 

Steven Rostedt was volunteered for the testing job by Clark Williams; Rostedt did not exactly disagree, noting that he had done that kind of thing in the past. Automating the realtime testing is something that needs to be done, he said. Ideally, each new stable kernel would be downloaded automatically, built, and run through a series of realtime-specific tests. Brown wryly noted that the next session in the microconference was on CI testing. He also said that it would make more sense to test the stable candidates, rather than the released kernels, so that any problems could be found before they get into the hands of users. 

At that point Kroah-Hartman popped up to say that the realtime kernel is not unique in any way; "you're special just like anybody else". He will take regression fixes into the tree as needed and can provide various ways to trigger the building and testing of the kernels for realtime. Rostedt agreed that realtime is not special in any way from the perspective of the stable maintainers; but the realtime developers need to work out how to automate their testing. 

Brown said that currently it is up to the RT-stable maintainers to apply the patches to a stable tree and manually test the resulting kernel. Kroah-Hartman suggested adding the realtime testing to the [KernelCI](<https://kernelci.org/>) infrastructure, so that it will be automatically built and tested whenever a stable candidate is released. Currently, the realtime patches are not merged into the stable tree right away, Rostedt said, because the stable changes often conflict with the realtime patches, but that should not be a problem once it is all upstream. 

Getting into KernelCI is "very easy", Kroah-Hartman said, but Brown noted that the kinds of testing that need to be done for realtime is different than for other parts of the kernel. The realtime tests have a performance criteria rather than a functional criteria, Williams said. But Kroah-Hartman said that KernelCI has both functional and performance testing now, so there should be no real barrier to adding the realtime tests. Brown agreed, but said that someone needs to get the tests into a form that fits into the infrastructure. 

As an example, Rostedt said that he runs a test that builds the kernel over and over again on multiple cores, while also running [hackbench](<http://manpages.ubuntu.com/manpages/xenial/man8/hackbench.8.html>) multiple times. All of that runs over a weekend, while he runs [cyclictest](<http://manpages.ubuntu.com/manpages/cosmic/man8/cyclictest.8.html>) with realtime tasks to record their latencies; he does not expect to find any latencies greater that 50µs. That kind of test would simply need to be packaged up and automated so that it can be run by bots of various sorts. 

Another question is whether realtime should have its own separate staging tree to try out new features, such as a new `futex()` interface, Rostedt said. Would it make sense to turn the current RT-stable tree into a "testing playground" for new features, he asked. If those features were deemed useful for the mainline, they could be backported to the stable kernels as well. But Williams wondered if it was time to "come back into the fold and not stay out in the cold"; he sees the value in an "RT-next" for development purposes, but does not think it would work well to support these features in earlier kernel series. While it did not come up in the discussion, those kinds of changes might also run afoul of the [stable kernel rules](<https://www.kernel.org/doc/html/v4.10/process/stable-kernel-rules.html>) about only fixing actual bugs. 

Rostedt more or less agreed with Williams but noted that there is a kind of "[catch-22](<https://en.wikipedia.org/wiki/Catch-22_\(logic\)>)" for API design, in that you cannot get a good API without users testing it, but that it is hard to get users to test without having a good API. Williams agreed that there is a problem there, but did not think backporting from RT-next would really help solve it—it is likely to just bring headaches for the realtime developers. Testers could build and use RT-next itself, he said. 

The main thing that needs to happen after realtime is in the mainline is to make sure there is a team paying attention to it going forward, Rostedt said. That team would ensure that realtime does not get broken in the stable kernels. Williams asked if there would be designated handlers for realtime bugs, but Rostedt thought that, once again, there is nothing special about realtime once it gets upstream. People will report bugs in the usual fashion, and the stable maintainers will direct the bugs to the realtime developers as needed. 

Now is a good time to get the automated testing in place, Sasha Levin said; it is more difficult to do that after the feature is in the mainline. Most of the RT-stable patches will apply automatically on the stable candidates at this point, Williams said, so those can be used to start working up the automated testing strategy. A plan soon formed to use Daniel Wagner's scripts for the 4.4-rt tree as a starting point to try to automatically merge the stable release candidates and the realtime patches; if that succeeds, then testing could be kicked off to see if there are any realtime-specific problems in the resulting kernel. Once realtime is in the mainline, the merging step would simply be dropped. 

#### Continuous integration

As the first session wound down, it segued nicely into a look at CI for realtime in the mainline led by Bastian Germann. There is some automated testing in place for realtime, he noted, though it was apparently not well known: the [CI-RT](<https://ci-rt.linutronix.de/RT-Test/>) system. It is a [Jenkins](<https://www.jenkins.io/>)-based CI system that is tailored to the needs of testing the realtime kernel. There is a one known lab running it at Linutronix (Germann's employer) on hardware donated by members of the [Linux Foundation Real-Time Linux project](<https://wiki.linuxfoundation.org/realtime/rtl/start>). 

Realtime developers can configure tests in CI-RT via a Git repository. The results of the tests are reported on the CI-RT site and also by email to the developer who is running them. The kernels are built on a build server, then booted on the target hardware, which serves as the first level of test. After that, the system runs tests somewhat similar to what Rostedt had described earlier. It uses cyclictest on both idle and stressed systems; the stress is created by hackbench coupled with other processes, such as a recursive `grep` that will generate a lot of interrupts, he said. The cyclictest results are then recorded for the systems. 

Once realtime gets into the mainline, the CI-RT system can be used as is, he said, just by reconfiguring the Git source being used. Beyond the mainline itself, there are some other trees that should get tested, including some that came out in the previous session, Germann said. The current release candidate for the mainline and linux-next should be tested; the stable kernels should be tested as well, including their release candidates as was discussed. The test frequency and duration will need to be established for each tree; for example, he suggested that linux-next could be tested for eight hours every night. 

No other CI systems currently run realtime tests, he said, though Brown wants to get them working in KernelCI. Germann said that more labs should be testing the realtime kernel once it gets merged. That will cover more hardware as well as raise the awareness of realtime among kernel developers. In order for that to happen, the realtime project needs to support other CI systems; KernelCI support is in the works, but he asked if there are other test or CI systems that should have support for realtime tests. 

After something of a digression into how to handle signing Git tags in an automated fashion, which was deemed undesirable, Nikolai Kondrashov suggested that CI-RT send its reports to KernelCI. He and others are working on [collecting and unifying test results](<https://foundation.kernelci.org/blog/2020/08/21/introducing-common-reporting/>) in a common database. 

Germann asked about the kinds of data that could be collected; ideally, CI-RT would want to present more than just a "pass" or "fail" and would include the latency measurements that were used to make that determination. Currently, the schema only provides a way to report the status of the test, Kondrashov said, but there is a way to attach additional data. The project is trying to work with the developers and operators of the various testing systems to determine what additional information should be added to the JSON schema. Veronika Kabatova mentioned that the Red Hat [Continuous Kernel Integration](<https://cki-project.org/>) (CKI) project would be willing to start running realtime tests, which would come with integration into the KernelCI unified reporting for free. 

Mel Gorman said that SUSE also runs a Jenkins-based CI system that uses some of the realtime tests as part of its performance testing. He had some suggested configurations for his [MMTests](<https://github.com/gormanm/mmtests>) that could be used to help with realtime testing. Those could be combined with hackbench or kernel compilation runs and cyclictest to determine if the realtime latency requirements are being met. It might make sense to integrate the realtime tests into some other existing testing client framework (such as MMTests), rather than trying to make multiple versions of those tests targeted at each different CI system, he said. 

The various CI efforts tend to congregate in the #kernelci channel on freenode or in the [automated-testing@lists.yoctoproject.org](<https://lists.yoctoproject.org/g/automated-testing/topic/mailing_list_for_board_farm/61241291?p=>) mailing list. Attendees plan to work with those groups to determine the right path forward in order get more CI testing for the realtime kernel. Once the realtime patches are finally merged, the CI-RT system should provide a good starting point for CI testing moving forward. 

As noted, these sessions were rather differently focused than most of those in the past. The final merging of the realtime patch set will make a big difference in how the project interacts with the rest of the kernel and the overall kernel ecosystem. It is important to get out ahead of the game with plans for stable-tree maintenance, along with ideas on how to make sure that the feature stays functional in the fast-moving mainline. The microconference would seem to have helped with both.
