---
title: Toward better testing
url: https://lwn.net/Articles/591985/
date: "March 26, 2014"
category: "Development tools-xfstests; Regression testing"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

Dave Chinner and Dave Jones started off the second day of the 2014 Linux Storage, Filesystem, and Memory Management Summit with a discussion of testing tools. What are we doing now, and what can be done better? The state of the art has improved considerably in recent times, but there are always ways to do better yet. 

Dave Chinner spoke as the maintainer of the xfstests filesystem testing suite. Despite its name, this suite has not been specific to the XFS filesystem for some time. There are, he said, more people now who are both using and contributing to xfstests, but there is still room for improvement. When a developer finds a filesystem bug, he said, the fix should include a contribution to the test suite to help ensure that the bug does not return in the future. 

James Bottomley asked how much code coverage is provided by xfstests now. It seems that the quality assurance people at Red Hat have done some coverage testing; about 75% of the code in the XFS filesystem is exercised by xfstests. Coverage of ext4 is a bit less at 65%; there are currently no tests to exercise the `ioctl()` code in particular. In general, the common code paths are tested well, but the more esoteric features lack test coverage. 

There was a request for the addition of power-failure testing to xfstests. Dave responded that there is a "crashme" script in xfstests now that can be used to randomly reboot the machine; XFS also has a special `ioctl()` that will immediately cut off the I/O stream, simulating a power failure on the underlying device. So, he said, there is no need to physically remove power to do power-failure testing; it can be done with the software tools that exist already. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Al Viro said that some tests will fail if the underlying storage partition is too small. Dave replied that there is a mechanism in the xfstests harness to specify how much space each test needs. In general, the minimum amount of space is 5-10GB; with that, most of the tests will run. At the other end, he runs some tests on a 100TB device, though, he noted dryly, it is wise to avoid any tests which need to fill the entire filesystem when working at that scale. Al also said that some tests can fail after thousands of operations; it would be nice for debugging to be able to replay an xfstests log and quickly zero in on places where things fail. 

In general, Matthew Wilcox said, it is not always easy to figure out why a specific test failed. Dave responded that this situation may not change; the purpose of xfstests is to alert developers that a bug exists, not to actually find that bug. He did say that he would accept patches that provide more hints to developers, but that there is also a reluctance to go back and change existing tests. It is easy to break the test itself, sending developers scrambling to find a filesystem bug that does not actually exist. Things are bad enough even without changing the tests, he said: every couple of years the GNU utilities developers feel the need to change the formats of error strings, causing problems in the test suite. 

Zach Brown complained that the discussion was focusing on details, when the most significant resource we have is the fact that Intel is paying people to put together testing infrastructure and actually run the tests on development kernels. Now, when developers introduce a bug, they will often get an automated email informing them of the fact. That is good, since, he said, the xfstests suite is painful to set up and run. 

Dave Jones asked if we need a similar test suite for the storage layer. Ric Wheeler responded that storage vendors have such suites, but those suites tend to be kept private. Mike Snitzer has a test suite for the device mapper; among other things, it helped to find problems with the recently merged immutable biovec work. When asked why this tool isn't more widely used, Mike responded that the fact that it is written in Ruby might have something to do with it. 

Another developer expressed a desire to coordinate filesystem tests with outside processes; the objective in particular was to create more memory pressure while the tests are running. Dave Chinner agreed that more testing should be done under memory pressure. Dave Jones suggested that the fault injection framework could be used; Dave Chinner agreed, but noted that fault injection, while exercising error paths, does little to exercise the reclaim paths in the kernel. So there is no substitute for real memory pressure. A program found in xfstests now will lock large amounts of memory into place, providing an easy way to add memory pressure to the system. 

Moving beyond xfstests, Dave Jones asked the community what kinds of tests are missing in general. James immediately responded that we need better ways of testing for performance regressions. Mel Gorman added that the community is "completely blind" when it comes to I/O performance. He has added some simple I/O tests to the mmtests suite and found some regressions in that area almost right away. But, he said, having the test is not enough, some kinds of problems require looking over the results in a detail-oriented fashion. Performance regressions may manifest themselves as latency spikes that have little effect on overall throughput numbers. 

Dave Jones recounted that, during the 3.10 development cycle, RAID5 was broken through the development cycle from the merge window until just before the release. Somebody, he said, should have found the problem sooner. It is also easy, he said, to bring down the kernel when assembling block devices with the device mapper. Developers, he said, simply are not trying to test a lot of this code in any sort of regular way. 

Ted Ts'o suggested that not enough developers have come to understand the deep sense of relief that comes from knowing that a set of changes has passed all of the regression tests. He wished he knew of a way to package that feeling and sell it to new developers. In the absence of that ability, he said, maintainers should do more yelling at developers who clearly have not run the available tests on their patches. Once a culture of regular testing sets in, it tends to become persistent. 

Dave Jones complained that, while we sometimes write tests for problems that have been experienced, we are not so good at proactively writing tests for functionality that might break sometime in the future. Dave Chinner agreed, saying that the quality assurance organizations run by distributors should really be writing more tests and trying to break things. In most organizations he has worked with, that kind of outside testing is the norm, but we don't do much of it in the kernel community. Developers, he said, tend not to break their own code well enough; we really need outside testers to find new and creative ways to break things. 

As the discussion wound down, there was some talk about areas that do not have good tests now. The filesystem notification system calls were mentioned. Some of the more obscure memory management system calls — `mremap()` or `remap_file_pages()` for example — don't have much test coverage. More test coverage for the NUMA memory policy code would also be helpful. Developers may eventually write these tests; hopefully others will then run them and let the community know when things break. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
