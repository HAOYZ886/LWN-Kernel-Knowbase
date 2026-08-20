---
title: On Linux kernel maintainer scalability
url: https://lwn.net/Articles/703005/
date: "October 12, 2016"
category: "Development model-Maintainers"
author: "By Jonathan Corbet October 12, 2016 LinuxCon
Europe"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 12, 2016

* * *

[LinuxCon Europe](<https://lwn.net/Archives/ConferenceByYear/#2016-LinuxCon_Europe>)

LWN's [traditional development statistics article for the 4.6 development cycle](<https://lwn.net/Articles/686697/>) ended with a statement that the process was running smoothly and that there were no process scalability issues in sight. Wolfram Sang started his 2016 [LinuxCon Europe](<http://events.linuxfoundation.org/events/linuxcon-europe>) talk by taking issue with that claim. He thinks that there are indeed scalability problems in the kernel's development process. A look at his argument is of interest, especially when contrasted with another recent talk on maintainer scalability. 

#### Beyond changesets merged

Sang's core point is that looking at the number of patches merged only tells part of the story; it says nothing about what had to happen to get those patches into the mainline. Looking at the last few years' worth of development cycles, he noted that relatively few patches carry tags beyond the Signed-off-by applied by the developer and the committer. In particular, around the 3.0 days, only about 20% of the patches in the mainline had an Acked-by, Reviewed-by, or Tested-by tag indicating that anybody other than the maintainer had seriously looked at them. That number is closer to 40% in current kernels, he said; it is a clear improvement, but still does not make him happy. For a properly scalable kernel process, he said, we should have much higher levels of review by developers who are not the subsystem maintainer. 

Another metric one can look at is the time difference between the date on the patch and the date on which it was first committed to a git tree. The Ethernet driver maintainers, he said, are heroes: 80% of all the patches were accepted within two weeks. A number of other subsystems do not do [![\[Wolfram Sang\]](https://static.lwn.net/images/conf/2016/lceu/WolframSang-sm.jpg)](<https://lwn.net/Articles/703006/>) anywhere near as well, and some have gotten significantly worse. I2C, Sang's own subsystem, has stayed about the same over the last three years, which surprised him. As the workload has increased, it has come to feel like things are getting much worse. 

The time-to-commit metric may be useful, but it is not without its flaws. The final version of a patch may have been committed fairly quickly, but previous versions could have languished without review for a long time. Patches that are rejected or that get lost are not considered at all. 

One way to try to get a better handle on things is to look at the [Patchwork](<http://jk.ozlabs.org/projects/patchwork/>) systems for the subystems that use it, and, in particular, to look at the backlog of patches found there. For [I2C](<https://patchwork.ozlabs.org/project/linux-i2c/list/>), it shows a relatively low backlog until about 3.16, when he gave up on trying to keep up with the flow and fell behind. The [ACPI subsystem](<https://patchwork.kernel.org/project/linux-acpi/list/>) has an amazing backlog of zero. The relevant maintainer (Rafael Wysocki) was in the room; he noted that it depends on how a subsystem uses Patchwork. He said that he quickly marks a lot of patches as inapplicable; Sang replied that he doesn't even have the time to do that. The [ext4 filesystem](<https://patchwork.ozlabs.org/project/linux-ext4/list/>) shows a linear growth in its backlog, up to about 800 patches currently. The numbers for several other subsystems were shown; almost all of them are going up. 

The problem, Sang said, is that the number of committers is not scaling to match the growing number of contributors to the kernel. We are getting more reviewers, but they are coming in slowly and are not anywhere near enough. As a result, the number of unprocessed patches is on the increase. 

How can this problem be addressed? Users can help by commenting on and, especially, testing patches. Developers need to be aware that sloppiness is often a problem; they should acknowledge when they have done suboptimal work. Developers need to take part in reviewing; if nothing else, they should review their own patches. For maintainers, working harder is not generally the solution; that just leads to burnout. They should get their tools in order and automate tasks whenever possible; looking at what other maintainers are using can be helpful. Companies should allow and encourage their developers to spend time reviewing patches. 

What he does not want to see is a "kernel infrastructure initiative". The [Core Infrastructure Initiative](<https://www.coreinfrastructure.org/>), run by the Linux Foundation as a way to channel resources to important but underfunded projects, is a good thing, but it is a reaction to a problem that got out of control. Things had to go wrong first. Sang would rather see action now to keep things from getting to that state. 

For I2C, Sang intends to step back a bit. He will become one of the I2C developers, one of its architects, and one of its reviewers, but he will not be the only one. That may slow things down in the short term, since he will be doing less patch review. The advantage is that he will stay sane, and will have the time and energy to try to address the problem on higher levels. 

#### The maintainer as bottleneck

While Sang intends to step back on patch review, his plan still calls for him to be the sole committer of patches for the I2C subsystem. In this context, it is interesting to look at another talk, given at [Kernel Recipes](<https://kernel-recipes.org/en/2016/>) one week earlier by i915 graphics driver maintainer Daniel Vetter. He, too, made the point that maintainers don't scale, but he would rather see maintainers get help at all levels. 

One year ago, he would have said that there was no problem in the i915 subsystem. Applying patches was relatively easy, after all. He had never reviewed the majority of the patches there; i915 has a number of developers who can do that. But, as the single maintainer, he gave the subsystem "a bus factor of one"; when he wasn't available for any reason, things simply came to a stop. 

At [the 2015 Kernel Summit](<https://lwn.net/Articles/662979/>), Linus Torvalds said that he has come to like the group maintainer model, where more than one person takes responsibility for a given subsystem. Vetter wanted to give that a try, but he quickly ran into a problem: nobody was willing to sign up as the co-maintainer for the i915 subsystem. He was, however, able to find developers who were willing to commit patches for i915; indeed, he signed up 15 of them. He figured he would experiment with the multiple-committer [![\[Daniel Vetter\]](https://static.lwn.net/images/conf/2016/recipes/DanielVetter-sm.jpg)](<https://lwn.net/Articles/703007/>) model for one release cycle. After all, nobody had ever really tried this before in the kernel, so it must be a stupid idea. 

That was one year ago, he said, and disaster has failed to materialize. Instead, he has "seriously happy contributors," and a whole set of reviewers who can apply the patches they look at. He is now "a bored maintainer," and all of the nagging and begging to get code merged has gone away. He has found that commit rights are a strong carrot that can be used to get developers and companies to contribute — and to be careful about the work they do. It also leads to "distributed conflict management" that makes life easier. 

So what does he do anymore? His main job at this point, as "the" maintainer for i915, is communications with the outside, including any work that requires coordination with other subsystem trees. He connects developers with the appropriate reviewers, and puts together the pull requests to send work upstream. And, of course, he "takes the blame for everything". 

To make this model work, he said, a subsystem clearly needs a team of developers, and non-maintainer reviews must be the norm. The group should be consistent, with developers who stay around; otherwise, enforcement via social feedback will not work well. Good documentation and tools are necessary; i915 has a set of process documents on [this page](<http://01.org/linuxgraphics/gfx-docs/maintainer-tools/drm-intel.html>). When somebody makes a mistake, if possible, a check should be put into the tools to keep it from happening again. 

Good testing is crucial to this model. A multi-committer tree can never be rebased, so there is no way to remove embarrassing mistakes. They really need to be avoided in the first place; that requires good pre-commit testing to ensure that the obscure corner cases do not break. 

The rough consensus model works best for a group like this. The default on any patch is "no action", so a developer's full disagreement will stop things. What's important, he said, is to have agreement on the goals for the subsystem; disagreement on the path taken toward those goals is acceptable. A good rule of thumb is "if you push a patch and there's screaming on IRC, you shouldn't have done it." 

In general, he said, the kernel could probably benefit from more maintainer groups like this. It is a more efficient way to maintain busy subsystems, especially those that currently have a lot of submaintainer trees. 

#### Meanwhile in Berlin

Fast-forward one week; your editor raised this idea in Sang's talk and asked whether the single-committer model might be part of the scalability problems raised there. The developers in that room tended toward skepticism over whether the idea could work outside of the i915 tree. Wysocki, in particular, seemed to feel that there were relatively few submaintainers who could be trusted with full commit access. These maintainers push patches that must be rejected fairly often, so they should not be able to commit directly to the subsystem tree. 

Perhaps these developers, too, would be pleasantly surprised if they were to run an experiment with more widely distributed commit rights. In any case, it seems likely that growing numbers of developers and patches will put more stress on subsystem maintainers. If those maintainers are not to become a choke point for kernel development, ways to spread the work they do will be required. 

[Your editor thanks both the Linux Foundation and Kernel Recipes for supporting his travel to these events.]
