---
title: Best practices for fstests
url: https://lwn.net/Articles/897061/
date: "June 7, 2022"
category: "Block layer-Testing; Filesystems-Testing"
author: "By Jake Edge June 7, 2022 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 7, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

As a followup to a [session on testing challenges](<https://lwn.net/Articles/896670/>) earlier in the day, Josef Bacik led a discussion on best practices for testing in a combined storage and filesystem session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). There are a number of ways that developers can collaborate on improving the testing landscape using fstests and blktests, starting with gathering and sharing information about which tests are expected to pass and fail. That information depends on a lot of different factors, including kernel version and configuration, fstest options, and more. 

#### Existing testing

Bacik began by briefly describing the testing setup that he uses for Btrfs, which continuously runs fstests. There is a dashboard web site that shows the test runs and failures, along with which configurations are affected. "That has been huge", he said; it has been running for a year at this point. 

He noted that Luis Chamberlain runs tests in a loop a thousand times, but Bacik's methodology is to simply run them once. Over the year they have been running that way, he has gathered a list of which tests are flaky; he then fixed many of the flaky tests to make them more reliable. Both types of testing are valid and useful, Bacik said, but his approach has given him the "best bang for the buck" for his needs. 

Ted Ts'o said that for finding bugs, he trusts fstests much more than "soaking in linux-next"; he only rarely finds bugs from linux-next testing. Fstests are "way more nitpicky", so a 15% failure rate on an fstest will correspond to a real bug, generally, but not one that users will encounter, even on the most stressful production workloads. That is part of the reason that he is not hugely concerned when there is an fstest failing sporadically. He would love to be able to fix all of the problems that he sees, but realistically, that is not possible; he does not have the time or the engineers on his team to do so. It is an "uncomfortable truth" that is difficult for new users of fstests to understand. 

Meanwhile, Ts'o said that he has files with lists of which tests he excludes, along with the reasons they are excluded. They also describe why the tests fail, which can be due to fstests bugs or simply flaky ext4 behavior. He would like to figure out where those files can be checked in so that others can benefit from that information. They are in his [GitHub repository](<https://github.com/tytso/xfstests-bld>), but perhaps adding them to the kernel documentation with a "freshness date" would make more sense. 

James Bottomley asked about the 0day testing bot for testing the linux-next kernels. Ts'o said that the bot only runs fstests for a single configuration for ext4, which generally runs cleanly. That configuration, which uses 4KB block sizes, is also the most-used configuration for production systems, but there are a total of 12 configurations that he tests. Bottomley suggested that making more use of the 0day bot would be worthwhile, but Ts'o worried about flaky tests causing lots of spurious regression reports "because the 0day bot got unlucky". But it is worth looking into, he said. 

Bacik said he would like to see the community "move towards this new reality" where it is easy to tell what tests are expected to fail. For example, he has no idea what tests are expected to fail for ext4, so when he makes a change that impacts ext4, he does not know if any failures are due to that change. Chamberlain has exclude files for [kdevops](<https://github.com/mcgrof/kdevops>), but it would be good to have a place where filesystem developers can obtain the exclude files and update them as needed. The tests listed in those files can also be useful as an exercise for onboarding new engineers, he said; they can be asked to track down why a test is failing. 

When he gets some time to do some fstests development, Ts'o said, he is going to add a mode that will immediately rerun a failed test 25 or 100 times to establish a failure percentage. He does not want to auto-exclude a test that is failing, say, 15% of the time because he will stop caring about it at that point; ext4 developers need those tests to continue to run. But there are others who simply want to try to determine if they broke anything with their patch, so there needs to be a way to address both types of testing. 

Omar Sandoval asked if it made sense to put these kinds of exclude lists directly into the fstests repository. Ts'o said that the lists are configuration specific; Chamberlain elaborated on that, noting that the lists depend on the kernel version, fstests configuration, and what type of underlying device is being used for the test. Tests on loopback filesystems can have different failure modes, for example. There was some discussion of the need to organize the lists based on all of the different factors. Agreeing on naming to describe the fstests configuration will be helpful to allow the associated exclude lists to be portable among various test runners. 

#### Standardization

Chamberlain thought that the kernel configuration could be standardized for test environments, as kdevops has a single configuration that can be used for KVM locally or for a variety of cloud providers. Ts'o said that [his test runner](<https://lwn.net/Articles/880916/>) (kvm-xfstests/gce-xfstests) also had a standardized configuration, so he and Chamberlain should compare notes. But there are still options that Ts'o needs when building kernels, for example enabling KASAN or kernel modules, which are needed when also running blktests. It is possible that the tool building Kconfig files should be standardized between the test runners, he said. 

Bacik said that he would like to see the filesystem developers get out of the business of running nightly and continuous tests. He currently has four systems at home that he uses to do that but would like to retire them and have the Linux Foundation or someone take over doing that job. Ts'o thought that even having a centralized dashboard to report the test results from various developers on their physical or cloud-based systems would be a step in the right direction; having one place to see the current state of fstests would be helpful. 

There is a problem with follow-through on these types of efforts, one attendee said. This topic comes up frequently at LSFMM, and ideas for solutions are discussed, but nothing really comes of it all. Christian Brauner agreed that follow-through has been lacking over the years. But Chamberlain said that kdevops came out of discussions at previous LSFMMs; he has spent a lot of time getting that working and it is available for use now. The only reason he did not use Ts'o's test runner is because he wanted to target multiple clouds; gce-xfstests only targets the Google Cloud. Kdevops already has support for exclude lists, Chamberlain said, which he updates regularly. "Kernel configuration that works on all cloud solutions, what else do we need?" 

Bacik agreed, noting that he had tried Ts'o's solution, but that it had not worked for Btrfs at the time, whereas kdevops does. He adopted it because he cannot sustain Btrfs development without community support for testing infrastructure; he had been trying to do test wrangling and Btrfs development without success. He would like to see more people coalesce around that solution, work out the bugs and kinks, then turn it over to someone else to either run it or to fund it running in the cloud. 

Chris Mason said that the Linux Foundation is not really set up to fund these kinds of efforts directly. Instead, they channel money from interested companies into projects of that sort. He said that it should just be a matter of getting the companies who are funding filesystem development to sign up. The funding is the easy part, he said; the hard part is to get everyone on the same set of tools. 

Ts'o said that there are actually two hard parts; the other is that there is a lack of engineering time to analyze the failures that occur. He would happily give Google Cloud credits to people who would run gce-xfstests if they would commit to spending time analyzing the failures that they find. There is perhaps a need to gather requirements, Ts'o said, because he has looked at kdevops and it does not address some critical requirements for his filesystem-development workflow. For example, kvm-xfstests can pick up a local kernel he just built on his laptop, toss it into a virtual machine, and run fstests on it, but kdevops targets the QA use case, so that kind of test is not supported. He thinks that unifying on things like kernel configuration and exclude-list handling would be a good place to start. 

Ts'o said that it may turn out that there are different tools for different use cases; the local kernel testing case is important for him. Bacik agreed that local kernel testing is critical, but thought it was possible that kdevops could add that capability. Chamberlain said that it should not be difficult to do so. 

Bacik said that he wanted to get his nightly testing systems switched over to using kdevops before the [Linux Plumbers Conference](<https://lpc.events/>) in Dublin in September, which is when many of the same people will be together in one room again. It was generally agreed that collaborating on requirements, kernel-configuration handling, fstests exclude-list handling, and things of that nature would continue, perhaps as threads on the fstests mailing list.
