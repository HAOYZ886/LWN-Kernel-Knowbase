---
title: "Kernel quality control, or the lack thereof"
url: https://lwn.net/Articles/774114/
date: "December 7, 2018"
category: "Development model-Kernel quality; System calls-copy file range"
author: "By Jonathan Corbet December 7, 2018"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 7, 2018

Filesystem developers tend toward a high level of conservatism when it comes to making changes; given the consequences of mistakes, this seems like a healthy survival trait. One might rightly be tempted to regard a recent disagreement over the backporting of filesystem-related fixes to the stable kernels as an example of this conservatism, but there is more to it. The kernel development process has matured in many ways over the years; perhaps this discussion hints at some of the changes that will be needed to continue that maturation in the future. 

While tracking down some problems with the XFS file cloning and deduplication features (the `FICLONERANGE` and `FIDEDUPERANGE` `ioctl()` calls in particular), the developers noticed that, in fact, many aspects of those interfaces did not work correctly. Resource limits were not respected, users could overwrite a setuid file without resetting the setuid bits, time stamps would not be updated, maximum file sizes would be ignored, and more. Many of these problems were fixed in XFS itself, but others affected all filesystems offering those features and needed to be fixed at the virtual filesystem (VFS) level. The result was a series of pull requests including [this one](<https://git.kernel.org/linus/4718dcad7decac3a43b7339b2226f3d987cca75c>) for 4.19-rc7, [this one](<https://git.kernel.org/linus/c2aa1a444cab2c673650ada80a7dffc4345ce2e6>) for the 4.20 merge window, and [this one](<https://git.kernel.org/linus/abe72ff4134028ff2189d29629c40a40bee0a989>) for 4.20-rc4. 

More recently, similar problems have been discovered with the [`copy_file_range()`](<http://man7.org/linux/man-pages/man2/copy_file_range.2.html>) system call, resulting in [this patch set](<https://lwn.net/ml/linux-fsdevel/20181203083416.28978-1-david@fromorbit.com/>) full of fixes. Once again, issues include the ability to overwrite setuid files, overwrite swap files, change immutable files, and overshoot resource limits. Time stamps are not updated, overlapping copies are not caught, and behavior between filesystems is inconsistent. Chinner's patch set contains another set of changes, almost all at the VFS level, to straighten these issues out. 

#### Mainline quality assurance

The discovery of these bugs has brought a fair amount of disappointment with it. It seems clear that these new features were not extensively tested before being added to the kernel; certainly no automated tests had been added to the xfstests suite to verify them. Dave Chinner [put it](<https://lwn.net/ml/linux-kernel/20181130214548.GO19305@dastard/>) this way: 

We ended up here because we *trusted* that other people had implemented and tested their APIs and code properly before it got merged. We've been severely burnt, and we've been left to clean up the mess made by other people by ourselves. 

In time, these bugs will be fixed and users of all filesystems should benefit. [Tests are being added](<https://lwn.net/ml/linux-xfs/154416379233.15344.4377231692638417530.stgit@magnolia/>) to help ensure that these features continue to work in the future. This is clearly a necessary effort; Chinner and Darrick Wong are performing a service for the kernel community as a whole by taking it on. This work does raise a couple of interesting issues, though. 

The first of those is the prospect of regressions in programs that have come to depend on the behavior of these system calls as it is supported in current kernels on specific filesystems. [Hyrum's law](<http://www.hyrumslaw.com/>) suggests that such users are likely to exist. Or, as Chinner [put it](<https://lwn.net/ml/linux-api/20181203081034.GW19305@dastard/>): ""the API implementation is so broken right now that fixing it is almost guaranteed to break something somewhere"". That could lead to some interesting discussions if users start to complain that kernel updates have broken their programs. 

#### Stable updates

The other question that arises concerns the backporting of all these fixes to the stable kernel updates; indeed, it was the [selection of one of the VFS fixes](<https://lwn.net/ml/linux-kernel/20181129060110.159878-25-sashal@kernel.org/>) for backporting that set off the current conversation. The XFS developers have long been hostile to the automatic inclusion of their patches in stable updates, feeling that the work to validate those patches in older kernels has not been done and that the risk of creating new regressions is too high. As a result, XFS patches are not normally considered eligible for backporting, but that exclusion does not extend to fixes at the VFS layer. 

In this case, Chinner [stated](<https://lwn.net/ml/linux-kernel/20181129224019.GM19305@dastard/>) that the current set of fixes has been validated for the mainline with a testing regime that runs billions of operations over a period of days; anything less risks not exposing some of the harder-to-hit bugs. Backporting those fixes to a different kernel would require the same level of testing to create the needed confidence that they don't create new problems, he said, and the XFS developers are too busy still fixing bugs to do that testing now. 

Chinner [followed up](<https://lwn.net/ml/linux-kernel/20181202232302.GT19305@dastard/>) with a lengthy indictment of the kernel development process as a whole, saying that it is focused on speed and patch quantity rather than the quality of the final result. The stable kernel process, in particular, is ""optimised to shovel as much change as possible with /as little effort as possible/ back into older code bases"". He pointed out that changes often appear in stable releases before they show up in a real mainline release (as opposed to an -rc release), which doesn't leave a whole lot of time for real stabilization. It is not, he feels, a small problem: 

I'm taking that one step further - what we are seeing here is the kernel community's systemic inability to address fundamental engineering process deficiencies because "speed and quantity" are considered more important than the quality of the product being produced. 

Sasha Levin [responded](<https://lwn.net/ml/linux-kernel/20181203092241.GC235790@sasha-vm/>) that the current process is the best that we can do at the moment: 

This is a case where theory collides with the real world. Yes, our QA is lacking, but we don't have the option of not doing the current process. If we stop backporting until a future data where our QA problem is solved we'll end up with what we had before: users stuck on ancient kernels without a way to upgrade. 

For the time being, the VFS and XFS patches will not be included in the stable kernel updates. Once the fixes are complete and the filesystem test suites have been filled out, Wong [said](<https://lwn.net/ml/linux-kernel/20181130203527.GP8125@magnolia/>), it should be possible to safely backport the whole set. At that point, this particular issue will be solved, but that is not likely to happen until after the 4.21/5.0 kernel release. 

For the longer term, there is still the problem that, as Wong put it: ""New features show up in the vfs without a lot of design documentation, incomplete userspace interface manuals, and not much beyond trivial testing"". One might well argue that this problem extends beyond VFS features. The kernel community has never had much of a process around the addition of APIs visible to user space; there are no real requirements to ensure adequate documentation, testing, or consistency between interfaces. The results can be seen in our released kernels, and in the API mistakes that just barely escape release because the right developer happened to notice them in time. 

Over the years, the kernel community has matured considerably in a number of ways. One need only look back to the days when we had no source-code management system, no rules on regressions, and no release-management discipline to see how much things have improved. The last few years have seen some big improvements around automated testing in particular. For all of our problems, the quality of our releases is quite a bit higher than it once was, even if it is not what it should be. Given time, it is reasonable to expect that we can build on that base to further focus our processes on the quality of the kernels we release, if that is something that the community decides it wants to do.
