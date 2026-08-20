---
title: "SCALE 8x: Ubuntu kernel development process"
url: https://lwn.net/Articles/376849/
date: "March 3, 2010"
category: Distributions
author: "By Jake Edge March 3, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
March 3, 2010

Canonical's kernel manager, Pete Graner, spoke at [UbuCon](<https://www.socallinuxexpo.org/scale8x/special-events/ubucon>)--held just prior to SCALE 8x--on the "Ubuntu Kernel Development Process". In the talk, he looked at how Ubuntu decides what goes into the kernel and how that kernel gets built and tested. It provided an interesting look inside the process that results in a new kernel getting released for each new Ubuntu version, which comes along every six months. 

Graner manages a ""pretty big"" group at Canonical, of 25 people split into two sub-groups, one focused on the kernel itself and the other on drivers. For each release, the kernel team chooses a "kernel release lead" (KRL) who is responsible for ensuring that the kernel is ready for the release and its users. The KRL rotates among team members with Andy Whitcroft handling Lucid Lynx (10.04) and Leann Ogasawara slated as KRL for the following ("M" or 10.10) release. 

The six-month development cycle is ""very challenging"", Graner said. The team needs to be very careful about which drivers--in-tree, out-of-tree, and staging--are enabled. The team regularly takes some drivers from the staging tree, and fixes them up a bit, before enabling them in the Ubuntu tree so that users ""get better hardware coverage"". 

Once the kernel for a release has been frozen, a new branch is created for the next release. For example, the Lucid kernel will be frozen in a few weeks, at which point a branch will be made for the 10.10 release. That branch will get the latest "bleeding edge" kernel from Linus Torvalds's tree (presumably 2.6.34-rc1), and the team will start putting the additional patches onto that branch. 

The patches that are rolled into the tree include things from linux-next (e.g. suspend/resume fixes), any patches that Debian has added to its kernel, then the Ubuntu-specific patchset. Any of those that have been merged into the mainline can be dropped from the list, but it is a ""very time-consuming effort"" to go through the git tree to figure all of that out. With each new tag from Torvalds's tree, they do a `git rebase` on their tree--as it is not a shared development tree--""see what conflicts, and deal with those"". 

The focus and direction for the Ubuntu kernel, like all Ubuntu features, comes out of the Ubuntu Developer Summit (UDS), which is held shortly after each release to set goals and make plans for the following release. Before UDS, the kernel team selects some broad topics and creates blueprints on the wiki to describe those topics. In the past, they have focused on things like suspend/resume, WiFi networking, and audio; ""a big one going forward is power management"", he said. 

The specifications for these features are ""broad-brush high-level"" descriptions (e.g. John has a laptop and wants to get 10 hours of battery life). The descriptions are fleshed out into various use cases, which results in a plan of action. All of the discussion, decisions, plans, and so on are captured on the [UDS wiki](<https://wiki.ubuntu.com/UDS-L>)

One of the longer kernel sessions at UDS looks at each kernel configuration option (i.e. the kernel `.config` file) to determine which should be enabled. New options are looked at closely to decide whether that feature is needed, but the existing choices are scrutinized as well. 

In addition, Graner said that the team looks at the patches and drivers that were added to the last kernel to see which of those should be continued in the next release. He pointed to [Aufs](<http://aufs.sourceforge.net/>) as a problematic feature because it always breaks with each new kernel release and can take up to three weeks to get it working. They have talked about dropping it, because Torvalds won't merge it into the mainline, but the live CDs need it. 

The kernel team has to balance the Ubuntu community needs as well as Canonical's business needs, for things like Ubuntu One for example, and come up with a set of kernel features that will satisfy both. The discussions about what will get in at UDS can get intense at times, Graner said, ""Lucid was pretty tame, but Karmic was kind of heated"". 

Lucid will ship with the 2.6.32 kernel which makes sense for a long-term support (LTS) release. 2.6.32 will be supported as a stable tree release for the next several years and will be shipped with the next RHEL and SLES. That means it will get better testing coverage which will lead to a ""very stable kernel for Lucid"". 

Each stable tree update will be pulled into the Ubuntu kernel tree, but LTS updates to the kernel will only be pushed out quarterly unless there is a "high" or "medium" security fix. For new kernel feature development, new mainline kernel releases and release candidates are pulled in by the team as well. Graner gave two examples of new development that is going on in the Ubuntu kernel trees: adding devicetree support for the ARM architecture, which will reduce the complexity of supporting multiple ARM kernels, and the AppArmor security module that is being targeted for the 2.6.34 kernel. 

Once the kernel version has been frozen for a release, the management of that kernel is much more strictly controlled. The only patches that get applied are those that have a bug associated with them. Stable kernel patches are "cherry-picked" based on user or security problems. There is a full-time kernel bug triager that tries to determine if a bug reporter has included enough information to have any hope of finding the problem--otherwise it gets dropped. One way to ensure a bug gets fixed, though, is to ""show the upstream patch that fixes the problem""; if that happens, it will get pulled into the kernel, Graner said. 

There are general freezes for each alpha, beta, and the final release, but the kernel must already be in the archive by the time of those freezes. Each time the kernel itself freezes, it ""takes almost a full week to build all of the architectures"" that are supported by Ubuntu. There are more architectures supported by Ubuntu than any other distribution ""that I am aware of"", he said. Each build is done in a virtualized environment with a specific toolchain that can be recreated whenever an update needs to be built. All of that means the kernel needs to freeze well in advance of the general release freeze, typically about a month before. 

Once the kernel is ready, it is tested in Montreal in a lab with 500 or 600 machines. The QA team runs the kernels against all that hardware, which is also a time-consuming process. Previously, the kernels would be tossed over the wall for users to test, but ""now Canonical is trying to do better"" by dedicating more resources to testing and QA. 

Managing kernel releases for a distribution is big task, and the details of that process are not generally very well-known. Graner's talk helped to change that, which should allow others to become more involved in the process. Understanding how it all works will help those outside of the team do a better job of working with the team, which should result in better kernels for Ubuntu users.
