---
title: Storage testing
url: https://lwn.net/Articles/789538/
date: "May 28, 2019"
category: "Development tools-blktests; Regression testing"
author: "By Jake Edge May 28, 2019 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 28, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Ted Ts'o led a discussion on storage testing and, in particular, on his experience getting [blktests](<https://github.com/osandov/blktests>) running for his test environment, in a combined storage and filesystem session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit. He has been adding more testing to his automated test platform, including blktests, and he would like to see more people running storage tests. The idea of his session was to see what could be done to help that cause. 

There are two test areas that he has recently been working on: NFS testing and blktests. His employer, Google, is rolling out cloud kernels for customers that enable NFS, so he thought it would be "a nice touch" to actually test NFS. He said that one good outcome of his investigation into running [xfstests](<https://github.com/kdave/xfstests>) for NFS was in discovering an [NFS wiki page](<http://wiki.linux-nfs.org/wiki/index.php/Xfstests>) that described the configuration and expected failures for xfstests. He effusively thanked whoever wrote that page, which he found to be invaluable. He thinks that developers for other filesystems should do something similar if they want others to run their tests. 

He has also recently been running blktests to track down a problem that manifested itself as an ext4 regression in xfstests. It turned out to be a problem in the SCSI multiqueue (mq) code, but he thought it would be nice to be able to pinpoint whether future problems were block layer problems or in ext4. So he has been integrating blktests into his test suite. Ts'o said that he realized blktests is a relatively new package, so the problems he ran into are likely to get better before long. Some of what he would be relating are his feedback on the package and its documentation. 

One of the biggest problems with blktests is that it is not obvious which tests are actually succeeding or failing. He put up a list of those tests that he thinks are failing, but he is not a block-layer specialist so it can be hard to figure out what went wrong. Some were lockdep reports that would seem to be kernel problems, but others may be bugs in his setup. It was quite difficult to determine which of those it was. 

For example, the NVMe tests were particularly sensitive to the version of NVMe being used. He found that the bleeding-edge, not-even-released version of the [nvme-cli](<https://github.com/linux-nvme/nvme-cli>) tool was needed to make some of the tests succeed. Beyond that, the required kernel configuration is not spelled out anywhere. Blktests requires a number of kernel features to be built as _modules_ or tests will fail, but it is not clear which ones. In a trial-and-error process, he found that 38 modules were needed in order to make most tests succeed. 

He plans to put his kernel configuration into xfstests so that others can use that as a starting point. It would be good to keep that up to date, Ts'o said. As these kinds of things get documented, it will make it easier for more people to run blktests. The code for his test setup is still in an alpha state, but he plans to clean it up and make it available; it is "getting pretty good" in terms of passing most of the blktests at this point. 

It is in the interests of kernel developers to get more people (and automated systems) running blktests, he said, as it will save time for the kernel developers. The way to make that happen is to find these kinds of barriers and eliminate them. Now that he has test appliance images that he can hand off to others to run their own tests on their patches, it makes his life easier. 

Ric Wheeler asked how many different device types were being tested as part of this effort, but Ts'o said that the NVMe and SCSI blktests do much of their testing using loopback. There are also tests that will use the virtual hardware that VMs provide. Wheeler said that there is value to testing physical devices that is distinct from testing virtual devices in a VM. Ts'o agreed that more hardware testing would be good, but it depends on having access to real hardware; he is testing on his laptop and would rather not risk that disk. 

Blktests maintainer Omar Sandoval said that the goal of blktests is to test software, not hardware, which is why the loopback devices are used. Some tests will need real hardware, while others will use the hardware if it is available and fall back to virtual devices or loopback if not. Wheeler noted that the drivers are not being tested if real hardware is not used. 

The idea behind this effort is to lower the barriers to entry so that anyone can test to see that they did not break the core, Chris Mason said. The 0-Day model, where people get notified if their proposed changes break the tests, is the right one to use, he said. That way, the maintainer does not have to ask people to run the tests themselves. 

Ts'o agreed that there should be a core set of tests that get run in that manner, but his current tests take 18-20 hours to run, which is not realistic for 0-Day or similar efforts. For those, some basic tests make sense. His plan is to ask people who are submitting ext4 patches to run the full set themselves before he considers them for merging. 

Wheeler said that there should be some device-mapper tests added to blktests as well. Sandoval said that the device-mapper developers have plans to add their tests, but that has not happened yet. Damien Le Moal agreed that specific device-mapper tests would be useful, but it is relatively straightforward to switch out a regular block device for a device-mapper target and run the regular tests. It is a matter of test configuration, not changing the test set; having a set of standard configurations for these different options would be nice, he said. 

Ts'o said that he has a similar situation with his ext4 encryption and NFSv3 tests; there is some setup and teardown that needs to be done around the blktests run. There is an interesting philosophical question whether that should be done in blktests itself or by using a wrapper script; xfstests uses the wrapper script approach and that may be fine for blktests as well. The important thing is to ensure that others do not have to figure all of that out in order to simply run the tests. Le Moal said that he had done some similar work on setup and teardown; he suggested that they work together to see what commonalities can be found. 

The complexities of setting up the user-space environment were also discussed. Luis Chamberlain noted that his [oscheck](<https://gitlab.com/mcgrof/oscheck>) project, which was also brought up in the [previous session](<https://lwn.net/Articles/789225/>), has to handle various distribution and application version requirements. He is using [Ansible](<https://github.com/ansible/ansible>) to manage all of that. 

Ts'o said that he builds a `chroot()` environment based on Debian that has all of the different pieces that he needs; it is used in various places, including on Android devices. There are some environments where he needs to run blktests, but the Bash version installed there is too old for blktests; his solution is to do it all in a `chroot()` environment. That also allows him to build his own versions of things like [dmsetup](<http://man7.org/linux/man-pages/man8/dmsetup.8.html>) and nvme-cli as needed. 

Ts'o uses Google Compute Engine for his tests, but Chamberlain would like to support other cloud options (e.g. Microsoft Azure) as well as non-cloud environments on other operating systems (e.g. Windows, macOS). He is planning to use [Vagrant](<https://github.com/hashicorp/vagrant>) to help solve that problem and is looking for others who would like to collaborate on that. Ts'o said that he believes the problem is mostly solved once you have the `chroot()` environment; there is still some work to get that into a VM or test appliance, but that is relatively minor. For his purposes, once it works with KVM, he is done, but he does realize that others have different requirements.
