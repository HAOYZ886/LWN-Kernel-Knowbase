---
title: "Is the kernel development process broken?"
url: https://lwn.net/Articles/126763/
date: "March 9, 2005"
category: "Development model-Kernel quality; Sucker tree"
author: 
---

According to some, the 2.6 development process has gone far out of control. Wildly destabilizing patches are routinely accepted, to the point that every 2.6.x release is really a development kernel in disguise. There are no more stable kernels anymore. As evidence, they point out certain high-profile regressions, such as the failure of 2.6.11 to work with certain Dell keyboards. 

It is true that the process has changed in 2.6, and that each 2.6 release tends to contain a great deal of new stuff. The situation is nowhere near as bad as some people claim, however. The problems which have turned up have tended to be minor, and most have not affected all that many users. Big, embarrassing security bugs, data corruption issues, etc. have been mostly notable in their absence. Kernel developers like Andrew Morton [don't think there is a problem](<https://lwn.net/Articles/126775/>): 

I would maintain that we're still fixing stuff faster than we're breaking stuff. If you look at the fixes which are going into the tree (and there are a HUGE number of fixes), many of them are addressing problems which have been there for a long time. 

Even so, there is a certain feeling that some 2.6 kernels have been released with problems which should not have been there. Last week, in an effort to improve the situation, Linus [posted a proposal](<http://lwn.net/Articles/125921/>) for a slight modification to the kernel release process. The new scheme would have set aside even-numbered kernel releases (2.6.12, 2.6.14, ...) as "extra-stable" kernels which would include nothing but bug fixes. Odd-numbered releases would continue to include more invasive patches. The idea was that an even-numbered release would follow fairly closely after the previous odd-numbered release and would fix any regressions or other problems which had turned up. With luck, people could install an even-numbered release with relative confidence. 

Over the course of a lengthy discussion, an apparent consensus formed: the real problem is a lack of testing. In theory, most patches are extensively tested in the -mm tree before being merged. -mm does work well for many things, and it has helped to improve the quality of patches being merged into the mainline. But the -mm kernels are considered to be far too unstable by many users, so they are not tested as widely as anybody would like. Even quite a few kernel developers work with the mainline kernels, since they provide a more stable development platform. 

The next step in the testing process is Linus's -rc releases. These kernels, too, are not tested as heavily as one might like. Many developers blame the fact that most of the -rc kernels are not really release candidates; they are merge points and an indication that a release is getting closer. Since users do not see the -rc kernels as true release candidates, they tend to shy away from them. For what it's worth, Linus [disagrees with the perception of his -rc kernels](<https://lwn.net/Articles/126781/>): 

Have people actually _looked_ at the -rc releases? They are very much done when I reach the point and say "ok, let's calm down". The first one is usually pretty big and often needs some fixing, simply because the first one is _inevitably_ (and by design) the one that gets the pent-up demand from the previous calming down period. 

But it's very much a call to "ok, guys, calm down now". 

The fact remains, however, that many people see a "release candidate" rather differently than Linus does. 

There are some -rc kernels which clearly _are_ release candidates; [2.6.11-rc5](<https://lwn.net/Articles/124946/>) is an obvious example. But even that kernel did not see enough testing to turn up the Dell keyboard problem. 

The real problem seems to have two components. The first is that widespread testing by users is a vital part of the free software development process. This is especially true for the kernel: no kernel developer has access to all of the strange hardware out there, but the user community, as a whole, does. The only way to get the necessary level of testing coverage is to have large numbers of users do it. But here is where the second piece of the puzzle comes in: most users are unwilling to perform this testing on anything other than official mainline kernel releases. So certain classes of bugs are only found after such a release takes place. 

A solution which was proposed was to bring back the concept of a four-number release: 2.6.11.1, for example. These releases would exist solely to deal with any show-stopper bugs which turn up after a major mainline release. Linus [was negative](<https://lwn.net/Articles/126782/>) about this idea, mostly because he didn't think anybody would be willing to do that work: 

I'll tell you what the problem is: I don't think you'll find anybody to do the parallel "only trivial patches" tree. They'll go crazy in a couple of weeks. Why? Because it's a _damn_ hard problem. Where do you draw the line? What's an acceptable patch? And if you get it wrong, people will complain _very_ loudly, since by now you've "promised" them a kernel that is better than the mainline. In other words: there's almost zero glory, there are no interesting problems, and there will absolutely be people who claim that you're a dick-head and worse, probably on a weekly basis. 

Linus went on, however, to outline how the process might work if a "sucker" were found who wanted to do it. The charter for this tree would have to be extremely restricted, with many rules limiting which patches could be accepted. The "sucker tree" would only take very small, clearly correct patches which fix a serious, user-visible bug. Some sort of committee would rule on patches, and would easily be able to exclude any which do not appear to meet the criteria. These conditions, says Linus, might make it possible to maintain the sucker tree, if a suitable sucker could be found. 

As it turns out, [a sucker stepped forward](<https://lwn.net/Articles/126785/>). Greg Kroah-Hartman has volunteered to maintain this tree for now, and to find a new maintainer when he reaches his limit. Chris Wright has volunteered to help. Greg released [2.6.11.1](<https://lwn.net/Articles/126324/>) as an example of how the process would work; it contains three patches: two compile fixes, and the obligatory Dell keyboard fix. [2.6.11.2](<https://lwn.net/Articles/126910/>) followed on March 9 with a single security fix. So the process has begun to operate. 

Greg and Chris have also put together [a set of rules](<https://lwn.net/Articles/126915/>) on how the extra-stable tree will operate. To be considered for this tree, a patch must be "obviously correct," no bigger than 100 lines, a fix for a real bug which is seen to be affecting users, etc. There is a new `stable@kernel.org` address to which such patches should be sent. Patches which appear to qualify will be added to the queue and considered by a review committee (which has not yet been named, but it ""will be made up of a number of kernel developers who have volunteered for this task, and a few that haven't""). 

The rules seem to be acceptable to most developers. There was one [suggestion](<https://lwn.net/Articles/126946/>) that, to qualify, patches must also be accepted into the mainline kernel. Being merged into the mainline would ensure wider testing of the patches, and would also serve to minimize the differences between the stable and mainline trees. The problem with this idea is that, often, the minimal fix which is best suited to an extra-stable tree is not the fix that the developers want for the long term. The _real_ fix for a bug may involve wide-ranging changes, API changes, etc., but that sort of patch conflicts with the other rules for the extra-stable tree. So a "must be merged into the mainline" rule probably will not be added, at least not in that form. 

How much this new tree will help is yet to be seen. It may be that its presence will simply cause many users to hold off testing until the first extra-stable release is made. This tree provides a safe repository for critical fixes, but those fixes cannot be made until the bugs are found. Finding those bugs requires widespread testing; no new kernel tree can change that fact.
