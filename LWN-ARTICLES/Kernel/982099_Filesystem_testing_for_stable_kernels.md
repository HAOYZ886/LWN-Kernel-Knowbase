---
title: Filesystem testing for stable kernels
url: https://lwn.net/Articles/982099/
date: "July 18, 2024"
category: "Filesystems-Testing"
author: "By Jake Edge July 18, 2024 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
July 18, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Leah Rumancik led a filesystem-track session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) on the testing needed to qualify XFS patches for the stable kernels. At last year's summit, Rumancik, Amir Goldstein, and Chandan Babu Rajendra [presented](<https://lwn.net/Articles/934941/>) on their efforts to test and backport fixes for the XFS filesystem to three separate stable kernels. There has been some longstanding [unhappiness in the XFS-development community with the stable-kernel process](<https://lwn.net/Articles/838819/>), which led to backports ceasing for that filesystem until Goldstein [started working on XFS testing](<https://lwn.net/Articles/896523/>) for the stable trees a few years ago. In this year's session, Rumancik updated attendees on how things had gone over the last year and wanted to discuss some remaining pain points for the process. 

#### Running tests

[ ![\[Leah Rumancik\]](https://static.lwn.net/images/2024/lsfmb-rumancik-sm.png) ](<https://lwn.net/Articles/982300/>)

She began by talking about the test runners for this work, focusing on [gce-xfstests](<https://docs.google.com/presentation/d/14MKWxzEDZ-JwNh0zNUvMbQa5ZyArZFdblTcF5fUa7Ss/edit#slide=id.p>), since a BoF for the other major test runner, [kdevops](<https://github.com/linux-kdevops/kdevops>), would follow her session. After more than a year of working with gce-xfstests, it is "way more obvious what the pain points are"; a lot of that is mechanical work that could be automated. The current setup performs hundreds of test runs on a baseline branch and multiple runs on the backports branch to try to detect differences. But flaky tests sometimes fail one time out of 20 or 100 runs, which means she has to go manually review and relaunch those tests, which is painful. 

For gce-xfstests, which is mostly what she uses for her testing, she would like to drop the baseline testing since it just adds to the noise. "If something is passing on the backports branch, we don't really care if it was passing or failing on the baseline branch." The end goal, then, would be to gather up failures or flakes on the backports branch and automatically submit them with a high run count to the baseline branch; gce-xfstests could compare the results between the two branches and output the tests that truly pass on the baseline and fail with the backports. 

The "expunge" lists, which are lists of tests that are expected to fail, need to be managed in some way, she said. Multiple people have their own lists for different filesystems, kernel versions, and so on. There have been suggestions to move those lists into fstests, which would make some sense, but there are still differences in the lists based on which test runner is used. Some centralization of the expunge lists would help, though it would be better to characterize the criteria when a test should not be run and use the `_notrun` mechanism to specify that. "So we all can benefit from it." 

XFS opts out of the backporting of patches that are chosen for the stable kernels, unlike most of the rest of the kernel does. But adding the "Cc: stable@vger.kernel.org" tag and using "Fixes:" tags on XFS patches is still useful to help the XFS stable testers find those patches, she said. 

There was some discussion last year about getting access to the patches that the [AUTOSEL patch-selection process](<https://lwn.net/Articles/825536/>) chooses for XFS, even though they will not directly go into the stable kernels. That would allow the XFS-stable team to sift through the patches to see which might make sense to pick up and test, Rumancik said. Another possible approach would be to use an automated-testing system on the stable -rc kernels to see if XFS problems are found; if so, the XFS AUTOSEL patches could be pulled and tested to see if the problem is fixed. 

"Currently I run a lot of tests." When she started work on the XFS testing, Darrick Wong gave her a list of test configurations to run. He suggested ten different XFS configurations and she has been doing 30 runs for each. The number of runs seemed excessive at the time, but it was a starting point that everyone could agree on; she thinks it is time to lower that number substantially, to perhaps five or even three. The number of configurations seems reasonable, since they cover various edge cases well. 

Goldstein asked how often she had found problems after 30 runs of the tests; "maybe once", she replied, from all of the backporting and testing. Steve French thought that some of the configurations should be tested more thoroughly than others, based on how commonly they were used in production; if bugs are found that affect particular configurations, the level of testing for those should increase. That kind of information can come from customers and product managers, which can then help reduce the amount of testing without affecting its effectiveness. 

Ted Ts'o said that ultimately it is up to the individual filesystem communities to determine how much testing is needed for patches going into the stable kernels. For ext4, he is not requiring that patches get tested before they are backported to the stable tree; instead, when he has time, he will do a single run of all of the ext4 configurations that he checks for problems. "That's good enough for me"; problems for ext4 have generally come from ongoing development and not from backports to stable. But the XFS developers have had a different experience and it is not up to others to decide how many test runs and which configurations are needed. 

Ritesh Harjani suggested that having a single run with lockdep enabled might help find more problems; Rumancik seemed to agree. Goldstein said that if she decided to change the testing regimen, she should just post it to the mailing list so that others have an opportunity to comment on it. Rumancik said that she was willing to write up a document that described the testing she is doing. 

#### Coverage

Dave Chinner said that optimizing the testing that is being done needs to be data-driven. The number of problems being found and how that changes depending on the number of test runs is one thing to look at. Another would be the additional coverage that comes from different configurations and test groups. For example, the "auto" group takes hours to run, while the "quick" group takes roughly half as long, but gives around 80% code coverage in his experience; the auto group only brings that up to 85%. 

It may be that the extra testing time is not worth it. Instead of 30 runs of the auto group, perhaps 30 runs—or even 20—of the quick group is sufficient. When a flaky test that fails one out of 20 times is discovered, he suggested trying to determine which of the test groups tickle it. The flaky tests are generally ones that are known to be non-deterministic and are often tagged as a stress test. It may well be that a subset of the auto group that runs even faster than the quick group is sufficient for 80% code coverage, and that is a good way to decide on which tests to run. 

French said that he has never really seen code-coverage analysis being applied to Linux filesystem testing over the years. He would like to have a way to generate a test group that is objectively better by running tests, gathering the code-coverage data, and removing any tests that do not add code coverage. 

Kent Overstreet said that he agreed; coverage analysis is not really being used. He has added code coverage to his test-running system but does not really look at the data frequently. He thinks that new patches should cause a look at that data to determine if there are new tests needed to accompany them. French said that the Samba project requires test cases with most patches, which would be a good habit to cultivate elsewhere. 

There is a need to be objective about tests, including the tests that we already have, French said; those that find bugs or regressions should be kept, and the same goes with those that increase the code coverage, but, otherwise, they should be removed. Chinner said that code coverage tells you some things, especially with regard to error-path coverage, but not everything. Some tests are done for correctness purposes and do not add any code coverage, but are still important. 

Wong said that fstests has the ability to gather the coverage information, which he has looked at. The coverage for running all of the tests for ext4 and Btrfs is in the mid-80s, he said, while XFS is in the high-80s. Iomap is around 91%, "which is better than I thought it would be, considering I wasn't even trying". Code-coverage support was added to fstests because Overstreet "nerd sniped" him into making the changes to fstests a year and a half ago, Wong said. 

As far as the number of tests being run, he had thought that 30 runs was a starting point for establishing a baseline and that "the number would go way down after that". He was surprised to find out that she is still doing 30 runs, "because I only ever do one". He has a "whole bunch of groups" that he runs every night and some "long-soak tests" that he runs for most of a week until a new -rc is released, which he rebases on and then restarts. 

Wong said that he also runs the tests in random order, which is "loads of fun". He wondered if Rumancik was doing any long-soak tests or just doing the 30 runs until they pass. Someone in the audience spoke up to say that the test order was not being randomized, but no one gave any indication that other, longer testing was being done for stable backports. 

#### Configuration management

Configurations for gce-xfstests are stored in separate files, but there has been some talk that it might make sense to centralize those under fstests for more widespread use, Rumancik said. The time it takes to run tests depends a lot on how large the test and scratch devices are defined to be, Ts'o said; each test-runner has its own opinions on those sizes, which impact the configurations. In addition, the type of hardware that the tests are targeting can also affect the run time of the tests. 

French said that he sometimes wants to append a particular mount option to a test configuration for a run; the configurations already have a set of mount options that go with the tests being run, but he wants to be able to add to that list. Rumancik said that she did not know of a way to do that, but she also cautioned that some tests do their own mount-option setting, disregarding those that are in the configuration. 

Ts'o said that mount options are handled differently by the various test runners, as well. He just goes into his and manually changes the options when he needs to. But the configuration handling is also different between test runners; his expunge lists have conditions (using `#ifdef`) based on Linux kernel versions where he knows there is a bug that will not allow a test to pass, but kdevops handles that differently. It might make sense to talk about how to centralize that into fstests, but for now, everyone has their own workflow and configuration-management strategy. 

There needs to be more thought on reducing the combinatorial explosion of things to test, Overstreet said, which amounts to "tech debt". Tests should not have to be run "N different times with N different configs"; the vast majority of those tests are not going to discover anything new. Specific, targeted tests would be more useful. 

"Test hardware is way cheaper than software-engineering time", Ts'o said. If he had ten developers to pay down the tech debt, he would do so, but that is not up to him; he can spin up a hundred virtual machines to run tests on fairly easily, however. 

It is well-known that "combinatorial explosions do not scale", Chinner said, but there is a certain amount that is required for filesystem testing. Options to `mkfs` and for mounting filesystems fundamentally change the behavior of the filesystem, thus they need testing. There is a need to choose configurations wisely, so that they do not completely overlap, but the combinations of those options still need to be tested. 

It is more than just filesystem developers who are running fstests, Harjani said; there are verification and support teams that are also running them. That is a reason that some amount of centralization of configurations for fstests is needed, which will make it easier for those people to run the tests. Chinner cautioned that it is hard to do that for different hardware; if the hardware does not support DAX, for example, those tests need to be excluded. As time expired, Harjani said that those kinds of differences could be accommodated in a centralized fstests configuration scheme; Chinner seemed skeptical but is interested in seeing patches to do so. 

A [YouTube video of the session](<https://www.youtube.com/watch?v=GMAYd7ckRwo>) is available.
