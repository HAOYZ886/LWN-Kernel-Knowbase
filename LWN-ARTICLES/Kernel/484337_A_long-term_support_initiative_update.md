---
title: "A long-term support initiative update"
url: https://lwn.net/Articles/484337/
date: "February 29, 2012"
category: "Civil Infrastructure Platform; Long-term support initiative"
author: "February 29, 2012 This article was contributed by Greg Kroah-Hartman ."
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

February 29, 2012

This article was contributed by [Greg Kroah-Hartman](<http://www.kroah.com/linux/>).

Back in October 2011, the Long-term Support Initiative (LTSI) was [announced](<https://lwn.net/Articles/464834/>) at the LinuxCon Europe event by Linux Foundation board member (and NEC manager) Tsugikazu Shibata. Since that initial announcement, a lot has gone on in the planning stages of putting the framework for this type of project together, and at the Embedded Linux Conference a few weeks ago, the project was publicly announced as active. 

Shibata-san again presented about LTSI at ELC 2012, giving more details about how the project would be run, and how to get involved. The [full slides for the presentation [PDF]](<https://events.linuxfoundation.org/images/stories/pdf/lf_elc12_shibata.pdf>) are available online. 

The LTSI project has grown out of the needs of a large number of consumer electronic companies that use Linux in their products. They need a new kernel every year to support newer devices, but they end up cobbling together a wide range of patches and backports to meet their needs. At LinuxCon in Europe, the results of a survey of a large number of kernel releases in different products showed that there are a set of common features that different companies need, yet they were implemented in different ways, pulling from different upstream kernel or project versions, and combining them in different kernel versions with different bug fixes, some done in opposite ways. [The results of this survey](<http://kernel.operationaldynamics.com/yaminabe/>) are now available for the curious. 

This is usually done because of the short development cycle that consumer electronic companies are under in order to get a product released. Once a product is released, the development cycle starts up again, and old work is usually forgotten, except to the extent that it is copied into the new project. This cycle is hard to break out of, but that needs to happen in order to be able to successfully take advantage of the Linux kernel community's releases. 

The LTSI project's goals were summarized as: 

  * Create a longterm community-based Linux kernel to cover the embedded life cycle. 
  * Create an industry-managed kernel as a common ground for the embedded industry. 
  * Set up a mechanism to support upstream activities for embedded engineers. 

Let's look at these goals individually, to find out how they are going to be addressed. 

#### Longterm community kernel

A few months ago, after discussing this project and the needs of the embedded industry, I announced that I will be maintaining a longterm Linux kernel for bugfixes and security updates for two years after it was originally released by Linus. This kernel version will be picked every year, enabling two different longterm kernels to be supported at the same time. The rules of these kernels are the same as the normal stable kernel releases, and can be found in the in-kernel file [Documentation/stable_kernel_rules.txt](<https://lwn.net/Articles/484478/>). 

The 3.0 kernel was selected last August as the first of these longterm kernels to be supported in this manner. The previous longterm kernel, 2.6.32, which was selected with the different enterprise Linux distributions' needs in mind, will still be maintained for a while longer as they rely on this kernel version for their users. 

As the 3.0 kernel was selected last August, it looks like the next longterm kernel will be selected sometime around August, 2012. I've been discussing this decision with a number of different companies about which version would work well for them, and if anyone has any input into this, please let me know. 

#### Longterm industry kernel

As shown at LinuxCon Europe, basing a product kernel on just a community-released kernel does not usually work. Almost all consumer electronics releases depend on a range of out-of-tree patches for some of their features. These patches include the Android kernel patch set, LTTng, some real-time patches, different architecture and system-on-a-chip support, and various small bugfixes. These patches, even if they are accepted upstream into the kernel.org releases, usually do not fit the requirements that the community stable kernel releases require. Because of that, the LTSI kernel has been created. 

This kernel release will be based on the current longterm community-based kernel, with a number of these common patchsets applied. Applying them in a single place enables developers from different companies to be assured that the patches are correct, they have the latest versions, and, if any fixes are needed, they can be done in a common way for all users of the code. This LTSI kernel tree is now public, and can be browsed at <http://git.linuxfoundation.org/?p=ltsi-kernel.git>. [Editor's note: this site, like the LTSI page linked below, fails badly if HTTPS is used; "HTTPS Everywhere" users will have difficulty accessing these pages.] It currently contains a backport of the Android kernel patches, as well as the latest LTTng codebase. Other patches are pending review to be accepted into this tree in the next few weeks. 

This tree is set up much like most distribution kernel trees are: a set of quilt-based patches that apply on top of an existing kernel.org release. This organization allows the patches to be easily reworked, and even removed if needed; patches can also be forward-ported to new kernel releases without the problems of rebasing a whole expanded kernel git tree. There are scripts in the tree to generate the patches in quilt, git, or just tarball formats, meeting the needs of all device manufacturers. 

During the hallway track at ELC 2012, I discussed the LTSI kernel with a number of different embedded companies, embedded distributions, and community distributions. All of these groups were eager to start using the LTSI kernel as a base for their releases, as no one likes to do the same work all the time. By working off of a common base, they can then focus on the real value they offer their different communities. 

#### Upstream support for embedded engineers

Despite the continued growth of the number of developers and companies that are contributing to the Linux kernel, some companies still have a difficult time of figuring out how to get involved. Because of this, a specific effort to get embedded engineers working upstream is part of the LTSI project. 

The LTSI kernel will accept patches from companies even if they contain code that does not meet the normal acceptance criteria set by Linux kernel developers due to technical reasons. These patches will be cleaned up in the LTSI kernel tree and helped to be merged upstream by either submission to the various kernel subsystem groups, or through the staging subsystem if the code will need more work than a simple cleanup. This feedback loop into the community is to help these embedded engineers learn how to work with the community, and in the end, be part of the community directly, so for the next product they have to create, they will not need to use the LTSI kernel as a "launching pad". 

#### Help make the project obsolete

The end goal of LTSI is to put itself out of business. With companies learning how to work directly with mainstream, their patches will not need the help of the LTSI developers to be accepted. And if those patches are accepted, their authors will not have to maintain them on their own for their product's lifetime; instead, they can just rely on the community longterm kernel support schedule for the needed bugfixes and security updates. However, until those far-reaching goals are met, there will be a need for the LTSI project and developers for some time. 

If you work for a company interested in working with the LTSI kernel as either a base for your devices, or to help get your code upstream, please see [the LTSI web site](<http://ltsi.linuxfoundation.org/>) for information on how to get involved.
