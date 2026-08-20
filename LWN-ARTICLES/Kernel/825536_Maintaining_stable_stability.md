---
title: Maintaining stable stability
url: https://lwn.net/Articles/825536/
date: "July 22, 2020"
category: "Development model-Stable tree"
author: "By Jake Edge July 22, 2020 OSSNA"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
July 22, 2020

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceByYear/#2020-Open_Source_Summit_North_America>)

The stable kernel trees are quite active, often seeing several releases in a week's time. But they are also meant to be ... well ... stable, so a lot of effort goes into trying to ensure that they do not introduce new bugs or regress the kernel's functionality. One of the stable maintainers, Sasha Levin, gave a talk at the virtual [Open Source Summit North America](<https://events.linuxfoundation.org/open-source-summit-north-america/>) that described the process of ensuring that these trees are carefully managed so that they can provide a stable base for their users. 

#### Background

The goals of the stable tree are somewhat in competition with each other, Levin said. The maintainers do not want to introduce regressions into the tree, but they also want to try to ensure that they do not miss any fixes that should be in the tree. It is "very tricky" to balance those two goals. The talk would follow the path of patches that fix bugs, from the time they are written until they get released in a stable tree, showing the mechanisms in place to try to ensure that only real, non-regressing fixes make it all the way to the end. 

The first stage is the [rules](<https://www.kernel.org/doc/html/latest/process/stable-kernel-rules.html>) for the kinds of patches that get accepted into the stable tree. They have to be small, straightforward fixes that are already upstream in Linus Torvalds's tree. No complex new mechanisms or new features are welcome in the stable tree. The patches have "passed the minimal bar" to get accepted into the mainline, but it is sometimes necessary for the maintainers (or patch submitters) to backport the patch. That is something the maintainers try hard to avoid, so that the testing of the mainline is effectively also testing everything in stable, but backports cannot be avoided at all times. If there are large, intrusive patches that must be backported—for, say, mitigations for speculative-execution processor flaws—the stable maintainers require a lot more testing, subsystem maintainer signoffs, and more to try to ensure that the backport is reasonable. 

#### Stable patch process

The stable process starts when someone is working on a patch. If they submit it upstream tagged for the stable tree, reviewers and maintainers will generally pay more attention to it. Because the patch will likely end up in users' hands quickly, it is important to ensure that the patch is correct. If a patch is submitted that fixes a problem, but is not tagged for stable, the subsystem or stable-tree maintainers may ask that it get tagged for stable and, perhaps, get a "Fixes:" tag to help with backports. 

[ ![\[Sasha Levin\]](https://static.lwn.net/images/2020/ossna-levin-sm.png) ](<https://lwn.net/Articles/826752/>)

There is a bot that grabs patches tagged for stable from mailing lists and tries to apply them to different stable trees based on the stable versions indicated or by using the Fixes: tag. If the patch does not apply correctly, the bot will alert the author to the problem and possibly offer suggestions for other patches that may need to be applied before the fix will apply. It is important to do this when the patch is still being worked on, Levin said, as developers are generally more responsive when the fix is still fresh in their minds. 

"There's never a bad time to send upstream." Unlike other types of patches, fixes can be sent to the mailing list at any point in the development cycle. There is no need to wait for a merge window or to target a particular release candidate. "You should never sit on a fix, if you think it is good to go", he said. 

Once a patch has been reviewed and gets accepted into a maintainer's tree, it will usually also end up in the linux-next tree, which means that it will get hit with a bunch of tests, mostly from bots of various sorts. Also, the [KernelCI](<https://kernelci.org/>) continuous-integration project will run its tests on various maintainer trees. These provide a "good first bar" for stable patches to clear. 

Another testing tree, stable-next, is created by pulling the patches in the linux-next tree that are tagged for stable but have not (yet) been merged into Torvalds's tree. The idea is that test failures in this tree are likely to be caused by patches that are making their way into the stable trees, so it raises the visibility of those kinds of problems. "We don't want those failures to be swallowed up by other failures in linux-next", Levin said. Doing so also helps find patches that are being fixed by later patches, which are not yet upstream; if a stable-tagged fix is actually buggy, then it can be held out of the stable tree until its fix gets committed to the mainline. 

#### Into the mainline

Once the patch is merged into the mainline, it will be exposed to even more tests, many of which are being run by kernel developers rather than bots. There is more exposure to different workloads and hardware at that point. The stable maintainers are appreciative of all of the work that people are doing testing the mainline as it helps make the stable trees better, he said. 

If a patch in the mainline does not have a stable tag but it looks like it may be a fix, it might get submitted to the AUTOSEL bot, which is a [neural network for finding fixes](<https://lwn.net/Articles/764647/>). AUTOSEL looks at various parts of a patch—its author, commit message, signoffs, files changed, and certain code constructs—in order to determine if the patch is a probable fix that is missing a stable tag. 

When a patch reaches the mainline, that is when the work of the stable maintainers truly begins, he said. Each patch that is taken into the stable trees is reviewed by one of the maintainers. They look at each patch manually to try to ensure that it is correct and that it is appropriate for the stable kernels. The AUTOSEL patches get an additional week of review time in order to hopefully weed out problem patches that were identified in that process. When a patch is queued for stable, an email is generated to the author informing them that it has been done so that they can object if they think it is inappropriate or being applied to the wrong tree. 

The stable maintainers try to avoid modifying patches that do not cleanly apply to a particular tree. Levin pointed to his [dependency-chain Git repository](<https://git.kernel.org/pub/scm/linux/kernel/git/sashal/deps.git/>) as a way to find a set of patches that will allow the unmodified fix to be merged. "We would rather take a few more patches to make a certain patch apply and build and test cleanly rather than modifying the patch", he said. Looking at the dependency chain will also often help find other fixes that did not get tagged for stable. 

Once a patch makes it into the queue for a stable release, it will get tested again by many of the same bots that test linux-next. These trees are generated frequently with new patches added, so any testing failures will often point to the latest patches that were added. Compared to linux-next, the stable queue trees have far fewer patches, so it is easier to sanity check them for missing patches, regressions, and so on. 

Once or twice a week, release candidates are tagged. That will generate "yet another mail" informing developers that their patches have made it into a stable release candidate. It gives developers another chance to object or comment on the patch with respect to the stable trees. The release candidates are also tested with real workloads, Levin said; users of the stable trees test them in their data centers and in their test farms. The tests are more comprehensive than those typically done for mainline release candidates, since they involve the "actual end users" of the trees, he said. 

Anyone who is concerned about regressions from the stable kernels is encouraged to get involved in this process by testing their own workloads with these kernels. The stable maintainers do not want to release kernels with regressions, so they want to hear about any problems from users. The process is fairly "free form", he said, so that companies who do not want to talk about their workloads publicly can still report problems they encounter privately to the stable maintainers. They will make every effort to address any problems found before any release so that regressions are minimized or eliminated. 

#### Aftermath

Once the stable kernel is released, there is still more that the maintainers do. They monitor the mailing lists for fixes and bug reports that may impact patches added to the stable kernels. When those are found, the maintainers move quickly to get them into the next stable release in order to reduce the amount of time any regressions or bugs stay in the stable kernels. 

One of the goals of the stable maintainers is to improve the testing and validation strategy for the kernel as a whole, Levin said. There is a belief that the stable kernels should only get a small number of changes "because it's stable". But if that were the case, it would mean that the tree is missing lots of important fixes. The way to address the problem is not by taking fewer patches, but "by beefing up the kernel's testing story". The maintainers work with projects like KernelCI and the [0-Day testing service](<https://01.org/lkp/documentation/0-day-test-service>) to help ensure that they are working well and have the resources that they need. 

Monitoring the downstream trees, like the kernel trees for Ubuntu and Fedora, is also something that the stable maintainers do. If a patch is in a distribution's kernel but is not in the upstream kernel, maybe it should be, especially if it fixes something. Similarly, they monitor the bug trackers of various vendors and distributions to spot fixes that may need to be added to the stable kernels. Recently, they have been looking at the older stable kernels to see if there are fixes that have been missed for them along the way; when those are found, they get added into those older stable trees. 

The stable kernel process has a lot of safeguards in place to try to ensure that regressions are not introduced into those kernels. The stable kernels are "way better tested" than the mainline because they are seeing actual real workloads, rather than "just developers trying it on their laptops". The [rate of regressions](<https://lwn.net/Articles/812231/>) is low, especially when compared with the mainline, he said. So people should feel confident to take each new stable kernel as it is released; in addition, there will never be fixes in older stable kernels that are not also in the newer stable kernels, so upgrading to a newer stable series will not introduce regressions—from the stable process, at least. 

[ ![\[Sasha Levin and conference UI](https://static.lwn.net/images/2020/ossna-levin-if-sm.png) ](<https://lwn.net/Articles/826753/>)

At the end of the talk, a somewhat differently dressed Levin appeared to answer questions that were submitted through the chat-like interface in the conference system. One asked about cooperation between the stable maintainers and projects like the [Civil Infrastructure Platform](<https://www.cip-project.org/>), which are maintaining kernels for longer time frames. Levin said there are patches flowing in both directions between the groups and that there is a lot of cooperation around KernelCI and other testing initiatives. In answer to another question, Levin said that he hoped that failure reproducers from [syzbot](<https://github.com/google/syzkaller/blob/master/docs/syzbot.md>) fuzz testing could be added as part of testing for the stable tree at some point. He also acknowledged that a "not for stable" tag might be needed in the future, though currently that is handled by a note in the commit message to that effect—hopefully along with the reason why. 

While the talk was interesting, it was still vaguely unsatisfying—virtual conferences unsurprisingly do not live up to their in-person counterparts. But that is the way of things for a while, at least, and perhaps beyond even the end of the pandemic. The carbon footprint of such gatherings is certainly of some concern. In any case, the stable kernel process seems to be in good shape these days, with attentive maintainers, lots of testing, and plenty of fixes to get into the hands of users. Levin's talk was definitely a welcome look inside.
