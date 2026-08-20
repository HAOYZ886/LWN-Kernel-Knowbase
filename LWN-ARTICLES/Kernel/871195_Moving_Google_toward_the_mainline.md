---
title: Moving Google toward the mainline
url: https://lwn.net/Articles/871195/
date: "October 5, 2021"
category: Google
author: "By Jake Edge October 5, 2021 OSSNA"
---

By **Jake Edge**  
October 5, 2021

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceByYear/#2021-Open_Source_Summit_North_America>)

Two Google engineers came to [Open Source Summit North America](<https://events.linuxfoundation.org/open-source-summit-north-america/>) 2021 to talk about a project to change the way the company creates and maintains the kernel it runs in its data centers on its production systems. Andrew Delgadillo and Dylan Hatch described the current production kernel (Prodkernel) and the problems that occur because it is so far from the mainline. Project Icebreaker is an effort to change that and to provide a near-mainline kernel for development and testing within Google; the talk looked at the project, its risks, its current status, and its plans. 

#### Prodkernel

Delgadillo began the talk with Prodkernel, which runs on most of the machines in Google's data centers; those systems are used for handling search requests and "other jobs both externally facing and internally facing", he said. Prodkernel consists of around 9000 patches on top of an older upstream Linux kernel. Those patches implement various internal APIs (e.g. for Google Fibers), provide hardware support, add performance optimizations, and contain other "tweaks that are needed to run binaries that we use at Google". Every two years or so, those patches are rebased onto a newer kernel version, which provides a number of challenges. For one thing, there are a _lot_ of changes in the kernel in two years; even if the rebase of a feature seems to go well, tracking down any bugs that crop up involves a "very large search space". 

[ ![\[Andrew Delgadillo\]](https://static.lwn.net/images/2021/ossna-delgadillo-sm.jpg) ](<https://lwn.net/Articles/871984/>)

There were some specific internal needs and timelines that drove the need for Prodkernel, he said, which is why Google could not simply use the mainline and push any of the extra features needed into that. He gave some examples of the features that are needed for Google's production workloads but that are not available in the mainline; those included a need to be able to set quality of service values from user space for outgoing network traffic, to have specific rules for the out-of-memory (OOM) killer, and to add a new API for cooperative scheduling in user space. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

One of the big problems with Prodkernel is that it detracts from upstream participation, he said. Any features that Google creates for production are developed and tested on Prodkernel, which can be as much as two years behind the mainline kernel. If the developer wants to propose the feature for the mainline, the Prodkernel model imposes two main hurdles. For one, the feature needs to be rebased to the mainline, which may be a difficult task when the delta between the versions is large. Even if that gets done, there is a question of the testing of the feature. The feature has been validated on Prodkernel with production workloads, but now it has been taken to a new environment and been combined with totally new source code. That new combination cannot be tested in production because the mainline lacks the other features of Prodkernel. 

Google's workloads tend to uncover bottlenecks, deadlocks, and other types of bugs, but the use of Prodkernel means that the information is not really useful to others. If Google is running something close to an LTS stable kernel, for example, reporting the bug might lead the team to a fix that could be backported; generally, though, finding and fixing the bugs is something that Google has to do for itself, he said. In addition, any fixes are probably not useful to anyone else since they apply to a years-old kernel. Also, any bugs that have been fixed in more recent kernels do not get picked up until either they are manually found and backported or the next rebase is done. 

The rebasing process is "extremely costly" because it takes away time that developers could be working on other things. Each patch has to have any conflicts resolved with respect to the upstream kernel; it may well be that the developer has not even looked at the code for a year or more since they developed it, but they have to dig back in to port it forward. Then, of course, the resulting kernel has to be revalidated with Google's workloads. Bugs that are found in that process can be difficult to track down. Kernel bisection is one way, of course, but conflicts from rebasing need to be resolved at every step; that could perhaps be automated but it still makes for a difficult workflow, he said. 

The delay associated with the rebasing effort worsens the problems with upstream participation, which makes the next rebase take that much more time. It is a pretty clear example of technical debt, Delgadillo said, and it just continues to grow. Each Prodkernel rebase increases the number of patches, which lengthens the time it takes to do the next one; it is not sustainable. So an effort is needed to reduce the technical debt, which will free up time for more upstream participation—thus further reducing the technical debt. 

#### Icebreaker

[ ![\[Dylan Hatch\]](https://static.lwn.net/images/2021/ossna-hatch-sm.jpg) ](<https://lwn.net/Articles/871987/>)

Hatch then introduced Project Icebreaker, which is a new kernel project at Google with two main goals. The first is to stay close to the upstream kernel; the idea is to release a new Icebreaker kernel for every major upstream kernel release. Those releases would be made "on time, we want to stay caught up with upstream Linux". That will provide developers with a platform for adding features that is close enough to the mainline that those features can be proposed upstream. 

The second goal is to be able to run arbitrary Google binaries in production on that kernel. It would be "a real production kernel" that would allow validating the upstream changes in advance of the Prodkernel rebase. Under the current scheme, Google has been "piling everything into the tail end of this two-year period", he said. With Icebreaker, that testing can begin almost as soon as a new mainline kernel gets released. 

Those goals are important because the team needs "better participation upstream". Developers working on features for kernels far removed from the current mainline have a hard time seeing the benefit of getting that feature upstream. There is a lot of work that needs to be done to untangle the feature from Prodkernel, test it on the mainline kernel, and then propose it upstream—all of which may or may not result in the feature being merged. The alternative is to simply wait for the rebase; time will be made available to do that work, but once the new Prodkernel is qualified, it is already too late for the feature to go upstream. 

Having kernels closer to mainline will also help Google qualify and verify all of the upstream patches that much sooner. Rather than waiting until the two years are up and doing a huge rebase and retest effort, the work can be amortized over time 

#### Structure

There are two sides to consider when looking at the structure of the Icebreaker effort, he said. On one side is how features can be developed in order to get them deployed on an Icebreaker kernel. On the other is how those patches need to be upgraded in order to get them onto a new mainline kernel for the next Icebreaker release. 

Icebreaker creates a fork from the mainline at the point of a major release. It adds a bunch of feature branches (also known as "topic branches") to that, each of which is a self-contained set of patches for one chunk of functionality that is being added by Google. That is useful in and of itself, because each of those branches is effectively a patch series that could be proposed upstream; "so you are starting with something upstreamable and not going the other way around", Hatch said. 

Development proceeds on those feature branches, with bug fixes and new functionality added as needed. Eventually, those feature branches get merged into subsystem-specific staging branches for testing. The staging branches then get merged into a next branch for the release. The next branch is an Icebreaker kernel that is "ready to go, but it still has its roots in these feature branches", he said. After the release is made, a "fan-out merge into the staging branches" is done, in order to synchronize them with the release version. Importantly, this fan-out merge is not done into the feature branches. Those stay in a pristine upstreamable state. 

By following the life of one of these feature branches, we can see how the upgrade process goes, he said. When a new mainline kernel is released, a new branch for the feature is created and the branch for the earlier kernel is merged onto it. The SHA1 values for the commits on the earlier feature branch are maintained and the conflict resolution is contained in the merge commit. 

Bug handling is easier with this workflow. The bugs can be fixed on the earliest supported feature branch where they occur and then merged into all of the successive feature branches. The SHA1 of the commit that introduced the bug and that of the fix will remain the same on those other branches. There is no need to carry extra metadata to track the different fix patches in each of the different supported kernel versions. 

The Icebreaker model is much more upstream-friendly than the current Prodkernel scheme, Hatch said. The Icebreaker feature branches are done on an up-to-date mainline kernel and they get tested that way, so the test results are more relevant to upstream. This will allow developers to propose features for the mainline far more easily. Much of the Icebreaker branch structure and the like can be seen in the diagrams in the [slides](<https://static.sched.com/hosted_files/osselc21/f1/OSS%202021%20-%20Icebreaker.pdf
>) from the talk. 

#### Risks

"There are some risks with Icebreaker, unfortunately", he said. One of the bigger ones is that there needs to be a lot of feature branch testing. There may be a tendency to treat those branches like a file cabinet, where patches are stored and merged into wherever they are needed. But that is not useful if it is not known whether "it builds or boots or passes any tests". 

Thus it is important to validate just the feature branch before merging it elsewhere. If it is known that it was working before the merge, then any subsequent breakage will have been caused by something in the merge. Otherwise, it is just complicating the whole process to merge a feature in an unknown state into a new tree. The same goes when upgrading to a new mainline kernel version, he said. 

The dependencies between features could be a risk for Icebreaker as well. The model is that features are mostly self-contained, but that is not completely true; there are some dependencies. They can range from APIs being used in another feature to performance optimizations that are needed for a feature to do its job correctly. That could be handled by resolving the dependencies on the staging branch, but those branches are not carried along to the next Icebreaker kernel, only the feature branches are. 

The answer is to do merges between feature branches, which does work, but adds some complexities into the mix. There is a need to figure out which branches can be merged into each other. "How crazy can we let these merges become?", he asked. There are no rules for when two feature branches should simply be turned into a single feature branch or when there is utility in keeping them separate; those things will have to be determined on a case-by-case basis. 

Another risk is that Icebreaker is much less centralized than the Prodkernel process is. Feature owners and subsystem maintainers within Google will need to participate and buy into this new workflow and model. They will need to trust that this new Icebreaker plan—confusing and complicated in the early going—will actually lead to better outcomes and a reduction in the technical debt. 

The last risk that Hatch noted is that features in Icebreaker do actually need to get upstream or it will essentially devolve into Prodkernel again. If more and more patches are added to Icebreaker without a reduction from some being merged due to features going upstream, the team will not be able to keep up with the mainline. The production kernel team needs to take advantage of the fact that Icebreaker is so close to mainline and get its features upstream. 

#### Status and plans

Delgadillo took over to talk about the status of Icebreaker. At the time of the talk, it was based on the 5.13 kernel, at a time when the 5.15 kernel is in the release-candidate stage. So the project is essentially one major release behind the mainline, which is "a lot closer that we have ever been". 

[ ![\[Dylan Hatch & Andrew Delgadillo\]](https://static.lwn.net/images/2021/ossna-delgadillo-hatch-sm.jpg) ](<https://lwn.net/Articles/871988/>)

In the process, some 30 patches were dropped from the tree because they were merged upstream. Out of 9000 patches being carried, 30 may not sound like a lot, he said, but it is a start. It is not something that would have happened without a project like Icebreaker. The team is working on 5.14 now and was able to drop 12 feature branches as part of that. Those were for features Google was backporting from the mainline, but that does not need to be done for recent kernels. That is another reduction in the technical debt, he said. Hopefully that process will get "easier and easier as we go along". 

In addition, issues have been found and fixed, then reported upstream or have been sent to the stable trees for backporting. That is not something that happened frequently with Prodkernel because it was so far behind the mainline. In general, they were build fixes and the like, he said, but were useful to others, which is a plus. 

Looking forward, Icebreaker plans to catch up to upstream at 5.15 or 5.16, which will be a turning point for the project. It will be "riding the wave" of upstream at that point, which will allow "us to relax the cadence at which we need to update our tree", he said. One of the problems that has occurred is that feature maintainers have had to rebase and fix conflicts every three or four weeks as Icebreaker worked on catching up; in the Prodkernel model, that would only happen every two years or so. Once the project has caught up, there will only need to be rebases every ten or so weeks, aligned with the mainline schedule. 

Testing the Icebreaker feature branches on top of mainline kernel release candidates is also something the project would like to do. That would allow Google to participate in the release-candidate process and help test those kernels. Once Icebreaker is aligned with mainline, it will make upstream development of features possible in a way that simply could not be done with Prodkernel. 

At that point, Delgadillo and Hatch took questions. The first was about the plans for Prodkernel: will there still be the giant, two-year rebase? Hatch said that for now Icebreaker and Prodkernel would proceed in parallel. Delgadillo noted that Icebreaker is new, and has not necessarily worked out all of its kinks. He also said that while Icebreaker is meant to be functionally equivalent to Prodkernel, it may not be at parity performance-wise. It is definitely a goal to run these kernels in production, but that has not really happened yet. 

Readers may find it interesting to contrast this talk with [one from the 2009 Kernel Summit](<https://lwn.net/Articles/357658/>) that gives a perspective on how things worked 12 years ago. 

[I would like to thank LWN subscribers for supporting my travel to Seattle for Open Source Summit North America.]
