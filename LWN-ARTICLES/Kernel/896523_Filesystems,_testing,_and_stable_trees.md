---
title: "Filesystems, testing, and stable trees"
url: https://lwn.net/Articles/896523/
date: "May 31, 2022"
category: "Development model-Stable tree; Filesystems-Testing"
author: "By Jake Edge May 31, 2022 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
May 31, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

In a filesystem session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Amir Goldstein led a discussion about the stable kernel trees. Those trees, and especially the long-term support (LTS) versions, are used as a basis for a variety of Linux-based products, but the kind of testing that is being done on them for filesystems is lacking. Part of the problem is that the tests target filesystem developers so they are not easily used by downstream consumers of the stable kernel trees. 

His interest in the problem comes about because he is using the 5.10 LTS kernel and the XFS filesystem. He realized that XFS is not being maintained in that kernel; there are only three XFS patches backported to it in the past two years or more. There is some [history](<https://lwn.net/Articles/838819/>) behind that, though most in the room already know it, he said. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2022/lsfmm-goldstein-sm.png) ](<https://lwn.net/Articles/896668/>)

He has been backporting XFS patches to 5.10 because there are more than just three bug fixes for XFS since that kernel was released. In something of a disclaimer, he said that it is his responsibility to do those backports; he is not suggesting that others should be doing that work. He has made some progress with the backports and has been doing some testing of them in conjunction with Luis Chamberlain. His intent in the session was to discuss the process for stable kernels and filesystems. 

One reason that the stable kernels exist, Goldstein said, is to allow multiple organizations to collaborate and "not duplicate work". That only works if the LTS releases are used by the "big players", so the value of those releases drops if they are not widely used. Many distributions do not use the LTS kernels, but there are some organizations that do. Google Cloud, for one, is following the stable kernel releases, and he has heard that Microsoft is doing the same. Android is also following the stable releases, but that project has no interest in XFS. 

The key to having stable kernels with stable filesystems is being able to run fstests (formerly xfstests) on them. That means collaborating on testing, the test suite, and the baselines of which tests are expected to pass and fail. Josef Bacik said that when he worked at Red Hat, one of the pain points was in running the most recent fstests on older kernels, as it would "blow up" in various ways, which was annoying. But running the latest fstests and seeing newer tests fail can also point to patches that you may want to backport "depending on how much pain you are willing to absorb", he said. 

Goldstein said that fstests are mainly used to test the upstream kernels; when they are applied to LTS kernels "things happen" so it is not easy to do so. Fstests is not friendly to people trying to test LTS kernels, which is a different approach than that of another test framework that he works with, the [Linux Test Project](<https://linux-test-project.github.io/>) (LTP). That project has some practices that could be adopted by fstests; in particular, having a standard way to annotate regression tests, giving the commit that fixed the bug and what version of the kernel it is fixed in. That way, if the test fails on a different kernel, "you get a hint" that maybe a backport of that commit is needed or, perhaps, that the kernel under test will not support the feature being tested. 

LTP also has a simple script that can be run on a kernel branch to determine if it has the commits that appear in the annotations, or has backports of those commits that refer to the original commits. That will give you a list of the tests that should work; the list will be customized to that exact kernel branch, he said. 

Ted Ts'o said that most filesystems are happy to allow the stable developers to choose fixes to incorporate—XFS is a notable exception to that. For ext4, that the process works well, he said; every year or so there is a problematic ext4 patch that has to be reverted from the stable trees because it was not suitable for them. Normally, those kinds of patches are spotted during the stable review. 

Ts'o and his team have been working on identifying XFS patches to apply to the 5.15 kernel, because that is a kernel of interest for Google, using the same scripts that Greg Kroah-Hartman and Sasha Levin use to identify candidate patches. It has taken longer to do this work than he had hoped, in part because of the time it has taken to get a baseline of which fstests should be passing so that they can detect failures caused by backports. They have been using an automated test system, with around ten different configurations based on input from XFS maintainer Darrick Wong. 

It turns out that there were some fstests that only passed if they cherry-picked some of the "hundred-odd out-of-tree commits" that are in Wong's personal fstests tree, but have not yet gotten to the upstream repository. So, Ts'o now has his own fstests branch with the pieces from Wong that were needed. 

It is his intent to report on the work that they have done to the XFS mailing list, including a list of the patches that they are proposing to add to 5.15. After that, there will need to be a negotiation about what is considered appropriate testing, Ts'o said, as well as a need to figure out how the XFS maintainers want to proceed. Whether the process will be to propose the fixes for stable and await any explicit nacks from the XFS folks, or whether the XFS maintainers will be explicitly choosing the set of patches to add to stable, is unclear at this point. That is a conversation that he hopes to have soon. 

Chamberlain said that in the past, the XFS maintainers have agreed that he and Goldstein could review XFS patches for the stable kernels. But, as noted by Ts'o and others, establishing the baseline takes a lot of thankless work; it also requires fairly large systems, Chamberlain said. Right now, each developer is making their best effort at testing, but the community needs to collaborate more on the testing effort; the next LSFMM session would cover some of that, he said. Candidates for XFS fixes can be sent to him and Goldstein; they will queue the patches up for their testing, which will help give some confidence about whether the patches are good candidates or not. 

Jan Kara came in over the Zoom link to say that the distributions, including SUSE where he works, do care about XFS fixes. The SUSE folks pick up XFS fixes and he thinks that Red Hat does the same thing. If those fixes do not end up in the stable kernel, they get backported to the enterprise kernels and then tested. The resources required to do all of that are fairly large. There is a need for developers with "at least a bit of a clue" to look at the patches to see if they make sense to be backported and then do that work if so. Then there is "quite a lot of testing", he said. 

Goldstein talked about a tool that he created when he was looking at all of the XFS fixes from 5.10 to 5.17, which turned out to be around 600 patches. The tool uses the public-inbox mailing list archives to collect up all of the relevant patch series and, in particular, the cover letter. That made it much easier to see what dependencies there are and which patches to choose. It is "still human work", but the tool is a great assistant. 

Ts'o noted that he does a round of testing of ext4 every three to four months using the latest LTS kernels. The resources required to actually run the test are modest; for a few dollars of Google Cloud time, he can run multiple configurations of fstests. The expensive part is the developer time to interpret the failures and to figure out if there is patch that did not get automatically chosen but should have. 

Every time he does that round of tests, he finds one to three patches that he needs to manually backport and send to the stable developers. He is not sure whether other filesystem maintainers are doing similar testing, but it is valuable. That kind of testing is also not something that the maintainers themselves would need to do, it might be a good opportunity to add some newer developers to the filesystem community, he suggested. 

There was some more discussion of what needs to be done to make it easier to run fstests on older kernels. Steve French wondered if there needed to be stable branches of fstests that could be kept in sync with the stable kernel releases. Goldstein said that annotations of commits and versions for fixes will be important to make it easier to use fstests on a wider variety of kernel versions.
