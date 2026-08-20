---
title: Rolling stable kernels
url: https://lwn.net/Articles/871989/
date: "October 6, 2021"
category: "Development model-Stable tree"
author: "By Jake Edge October 6, 2021 OSSNA"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
October 6, 2021

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceByYear/#2021-Open_Source_Summit_North_America>)

Sasha Levin, one of the maintainers of the stable kernels, gave a presentation at [Open Source Summit North America](<https://events.linuxfoundation.org/open-source-summit-north-america/>) 2021 on a proposal for a different way to handle the stable tree. He noted that throughout most of the kernel's history, version numbers did not really mean anything, but that the versioning scheme suggests that they do, which leads to a disconnect between how the kernels are seen versus how they are actually maintained. He proposed making a "rolling stable" release that provides users what they need—timely fixes to their kernel—without forcing them to choose to switch to a new version number. 

#### Context

He began with the history of kernel versioning schemes and how they have evolved over the years. Starting with version 0.01, Linus Torvalds used a common version-number mechanism that is adopted by smaller projects: if there were just a few small features, bump the number a little bit, if there were bigger features or more of them, bump it by more. When Torvalds decided he had enough of a critical mass that it was ready for wider use, he released Linux 1.0. 

[ ![\[Sasha Levin\]](https://static.lwn.net/images/2021/ossna-levin-sm.jpg) ](<https://lwn.net/Articles/872063/>)

Once 1.0 was out, Linux got more users and those users were not necessarily kernel developers; they wanted to develop applications and run their own workloads. They did not want to run on a development version of the kernel, which is where the idea of a stable kernel came from. The kernel proceeded with a model where one branch was for the stable kernel and other was for doing development. 

Fixes were backported from the development branch to the stable branch; once the development branch was ready for release, it would become the next stable branch. The kernel continued with the concept that versions were bumped for new features but the lack of central management meant that it was not clear when a release was ready to be made. In the period between the 2.0 and 2.6 kernels, the process kind of ground to a halt, Levin said. 

The process of going from 2.4 to 2.6 was "an adventure that took three years". That led the community to recognize that the process was not working and that eventually "it would take an infinite amount of time to go to the next version", he said. So, during the 2.6 era, the kernel developers came up with a new approach; instead of feature-based releases, they would switch to a time-based release process. 

Eventually, the idea that evolved was that new kernels would be released every ten weeks. The kernel developers also started saying that version numbers did not matter—and they really didn't. But that is an example of "engineers doing marketing", Levin said, because the version-numbering scheme did not change, nor did the lack of meaning for version numbers get clearly communicated to customers. 

That led to the now-familiar two-week merge window, followed by seven or eight release candidates, then a release. Every kernel was considered stable, so there was no separate development kernel. Some customers wanted more stability, which is where the idea of long-term stable (LTS) releases came from, he said. This model worked so well that the kernel stayed on 2.6.x for roughly eight years. 

#### Expectations

But mistakes were made in switching to the new time-based development process. While kernel developers ignored version numbers, customers did not, and the community did not sell the idea of time-based versions well. Since the numbers were still present, customers assumed they meant something, so a move from, say, 3.20 to 4.0 was "big and scary" because the major version had changed. 

Staying on 2.6.x for so long also had some ramifications. It was done in part to help customers get used to the new system, but to some extent, they got too comfortable. When the kernel developers wanted to move to 3.0, they found that various assumptions about the kernel version were baked into some products, so workarounds had to be added at that time. 

Customers were getting used to minor-version updates (e.g. from 2.6.y to 2.6.y+1), but they started to make assumptions about their meanings too, Levin said. Once the minor number got large, they treated it as a "more minor" change. In their eyes, 2.6.1 to 2.6.2 was an important change that needed an immediate upgrade, but 2.6.30 to 2.6.31 was a lesser change that could be deferred until 2.6.35 or something. Of course, to the kernel community, there was no difference. 

To address that, the plan was made to move to 3.0 and to increase the second number (minor version) with each release, effectively dropping the use of the third element of the version number for mainline releases. Levin called this the "finger and toes" version-number scheme, because Torvalds said that he could count up to around 20 "minor" releases before running out of digits and needing to increase the major version number. So after 3.19, 4.0 was released. 

To kernel developers, version numbers continued to have no meaning other than as a measure of where things were on the timeline, he said; users were encouraged to run the latest version. But if you tell customers they need to move from 3.20 to 4.0, they will consider it a multi-year effort that requires planning and paperwork; "so they will only do it when they have time, which is never". Going from 3.20 to 4.0 seems like a big jump, and 4.0 to 4.2 seems much smaller, even though the latter brings roughly double the number of changes. 

The kernel community has gotten better over time at testing, avoiding performance regressions, not breaking user space, and other things that should make it easier for users to migrate easily. But there is still resistance to doing so, in part because the kernel developers have not sold users on the plan. 

Some distributions and other users have listened, however, and are keeping up with the latest kernels or the LTS versions. They have recognized that the releases are simply time-based, so upgrades are not actually all that scary. Beyond that, users want the latest shiny features ("BPF, the new I/O scheduler, they want all the performance they can get") and they want all of that immediately. They do not want backports of more recent features onto older kernels. They also want frequent security updates. All of that aligns well with what the kernel developers want—and deliver. 

Enterprise kernels are less popular these days in part because "maintaining an ancient kernel has become exponentially difficult". The kernel moves so fast that there are more and more backports that need to be done, each of which complicates the maintenance of the enterprise kernel. In practice, most of the enterprise kernels have become "frankenkernels"; they contain backports of the majority of the current kernel and the version number they claim is not at all accurate. 

#### Rolling

In the "LTS world", users would pick a kernel LTS series, say 5.10.x, and keep upgrading their kernels to the latest in that series until that became impossible. When that happened, they would pick a new LTS series, switch to that, and continue from there. Kernel developers have been trying to sell the idea that they should switch sooner; when the next release comes out (in this case, 5.11.x), they should start using that series. 

But making that switch takes a conscious effort by the user; they also have to pick a time but "there's never a good time to say 'we're changing our kernel', especially this often", he said. Beyond that, there are some technical hurdles to switching to a new kernel Git branch mid-stream, there may be patches that need to be forward-ported, and so on. 

So, Levin asked, "what can be done to improve this process?" The kernel community provides the same guarantees no matter whether the change is between major versions, minor versions, or stable versions; "it promises that it will not break user space". It might break user space by accident, but it will fix the kernel if it does. Those breakages can occur on any kind of upgrade, though; no upgrade type is more or less prone to accidental breakage. "Version numbers still, really truly don't matter". 

The idea would be to "eliminate the friction point" where customers need to choose to move to the next series. Currently, the stable and LTS trees have a branch for each mainline release; effectively, the stable trees get forked every time Torvalds does a release. The mainline Git tree just continues on; once the next merge window opens, patches bound for the next release get committed to the main branch. There is no equivalent mechanism for the stable tree. 

The stable maintainers want to create a different structure that eliminates the need for users to switch branches each time they move to a new series. It would provide the same guarantees as the current trees and run on the same schedule. But it would roll forward by default, as does Torvalds's tree. 

The idea is to use "Git magic" to make it all work; there is no cherry picking of patches. Instead a "hack on Git merges" is used, which preserves the SHA1 values as they are in the existing stable branches. The Git tags survive and the SHA1 preservation means that the folks doing verification to prevent supply-chain attacks can do so more easily. Bisection still works, authors don't change so Git blame works, and so on. The stable maintainers just "manipulate Git a little bit to give the appearance that all this development happened in a single branch", he said. It is simply a change in "how we present kernel trees to the end user". 

Currently, users will be on _a_ stable branch or _an_ LTS branch, but under this model, they would instead be on _the_ stable or LTS branch. They are not using some random kernel version, but are instead using the version that the kernel developers have recommended for the best experience. From that, they will get all of the latest features, bug fixes, and security updates. 

#### Concerns

There are, of course, some concerns about this approach that he has heard in feedback from companies and users. The first is that there is an increased need for testing under the rolling model that is being pushed onto the users. But he disagrees. Currently users think they do not need to retest much if they go from x.y.50 to x.y.51, but that is not true. The same is true with rolling. Testing must be done whenever the kernel changes. 

Another area of worry is that the rolling model effectively "masks the merge window inside of it". One day the rolling stable tree might be on 5.7.15 and the next on 5.8.1; within that pull are a lot of patches that are from the 5.8 merge window, which is a big step. But it all goes back to testing, he said; it is possible that the merge window introduced a lot of bugs, but it is also possible that a single stable patch does the same thing. 

The schedules are staying the same, he said. The rolling stable and LTS branches are just derivatives of the current branches of the stable tree. And the new model does not require users to do anything on a different schedule; it simply gives them the latest kernel when they decide to upgrade and do a Git pull. 

The rolling model is not a radical departure, nor is it a new idea. "If you are using Debian, that's kind of how it works"; the Debian testing kernels are more or less the equivalent of the rolling stable branch, he said. He is confident that the rolling model will work because there distributions and devices that already use the model in some form; "they like how it works". Overall, it is an attempt to address some of the mistakes made in the past around versioning and what the stable developers told users about what they should use, Levin said. 

The [rolling stable](<https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/log/?h=linux-rolling-stable>) and [rolling LTS](<https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/log/?h=linux-rolling-lts>) branches already exist; rolling stable is at 5.14.9, as of this writing, while rolling LTS is at 5.10.70. As mentioned, they are just derivatives of the existing stable and LTS branches; there is nothing in them that is not in those other two. "If you want to follow the development philosophy of the Linux kernel, these branches are going to make it super easy for you". 

In answer to a question, Levin said that there is some time between a mainline release and the corresponding changes ending up on the rolling stable tree. Once 5.15 is released, they will wait until after the 5.16 merge window closes before incorporating the changes from 5.15. 

He was asked about the future of the LTS releases, as well. He noted that the LTS idea is something of a contradiction; on one hand, the stable maintainers recommend always using the latest LTS, but on the other, LTS releases are supported for five years. His personal hope is that LTS releases go away over time, but he does not realistically see that happening anytime soon because there is not enough testing and qualification being done. He expects, though, that in 20 years there will be no LTS kernels. 

[I would like to thank LWN subscribers for supporting my travel to Seattle for Open Source Summit North America.]
