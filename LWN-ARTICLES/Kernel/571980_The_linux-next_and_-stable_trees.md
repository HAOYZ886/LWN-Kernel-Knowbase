---
title: "The linux-next and -stable trees"
url: https://lwn.net/Articles/571980/
date: "October 29, 2013"
category: "Development model-Stable tree; Development model-linux-next"
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

While most kernel development focuses on the mainline, Linus's tree is not the only one out there. Two 2013 Kernel Summit sessions focused on a couple of the other important trees: linux-next and the stable tree; coverage of both those sessions has been combined into this article. 

#### linux-next

Linux-next maintainer Stephen Rothwell attended the Kernel Summit as the last stop in a long journey away from the office — and from maintenance of the linux-next tree. That tree continued to function in his absence, the [![\[Stephen Rothwell\]](https://static.lwn.net/images/conf/2013/lce-ks/StephenRothwell-sm.jpg)](<https://lwn.net/Articles/572015/>) first time it has ever done so. Stephen's session covered the current state of this tree and how things could maybe be made to work a little better. 

He started by thanking Thierry Reding and Mark Brown for keeping linux-next going during his break. Linux-next is a full-time job for him, so it has been hard to hand off in the past, but the substitute maintainership appears to have worked this time. 

Stephen routinely looks at the code that flows into the mainline during merge windows to see how much of it had previously appeared in linux-next. For recent kernels, that figure has been approaching 90%; it would probably be difficult, he said, to do better than that. What might be improved a bit, though, is how _long_ that code is in linux-next. In general, only 72% of the code that appears in linux-next is there one week before the merge window opens; that figure drops to less than 60% two weeks prior. So he tends to be busy in the last couple weeks of the development cycle, dealing with lots of merge conflicts and build failures. It leads to some long days. It would, he allowed, be nice of more of that code got into linux-next a bit earlier. 

There are, he said, 181 active trees merged into linux-next every day. Some of those are second-level trees that, eventually, reach mainline by way of another subsystem maintainer's tree. 

The worst problem he encounters is whole-tree changes that affect files across the kernel. Those create a lot of little messes that he must then try to clean up. It would be nice, he said, to find a better way to do things like API changes that require changes all over. Linus added that the reformatting of files to fix coding-style issues before applying small changes is "really nasty," leading to "conflicts from hell." It would be better to separate things like white-space changes from real work by a full release — or to not do the trivial changes at all. Ben Herrenschmidt suggested that white-space changes could be done at the end of a series, making them easy to drop, but Linus said that doesn't help, that the conflicts still come about. The best way to do white-space changes if they must be done, he said, is to do them to otherwise quiescent code. 

Stephen said that he still sees more rebasing of trees than he would like; rebasing should be avoided whenever possible. Grant Likely asked about one of the common cases for rebasing: the addition of tags like Acked-by to patches that already appear in linux-next. Linus said that this is a bit of a gray area but that, in general, if those tags do not show up in a timely manner, it's usually best not to bother with them. Stephen added that the addition of tags to patches in a published git tree may be an indication that the tree has been published prematurely; such trees should not be fed into linux-next until they are deemed ready for merging. 

James Bottomley pointed out that developers often publish git trees to get attention from Fengguang Wu's automated build-and-test system. But such trees do not need to be fed into linux-next to get that kind of testing; they can be put out on a separate testing branch. Ingo Molnar said that some tags, like Tested-by, can arrive fairly late; we really do not want to drop them and fail to credit our testers. Ted Ts'o added that employers often count things like Reviewed-by tags and that it is important to get them in. 

Linus's response was that timeliness matters; a too-late review is essentially useless anyway. That said, he also said that some people do take the "no-rebase" policy a little bit too far. There are times when rebasing a tree can be justified, especially if other developers are not basing their own trees on the rebased tree. For example, when a patch turns out to be a bad idea, it can be better make it disappear from the history and to "pretend that all crap just didn't happen." Ben added that he asks his submitters to base their trees on the mainline; then he can rebase his tree and they will not be adversely affected by it. 

Dave Jones complained about bugs which are claimed to be "fixed in linux-next," but those fixes then sit in linux-next for months until the next development cycle comes around. The right response to that problem, Andrew Morton said, was to "steal them" and forward them directly to Linus for immediate merging. 

In general, Linus said, he has been using the linux-next tree a lot more than he used to because it is working a lot better. It has become a good indication of what will be coming in the next merge window. In general, the group agreed that linux-next is a valuable resource that has done a lot to make the development process work more smoothly. 

#### The stable tree

Stable tree maintainer Greg Kroah-Hartman gave a quick update on the state of the various stable trees he runs. One of the biggest changes in stable tree maintenance, he said, is that he is starting to delay the inclusion of patches until they have appeared in one of Linus's -rc kernels; that delay will typically be one or two weeks. There have been a few incidents recently where "stable" patches have caused regressions; Greg hopes that, by inserting a small delay, he can flush out the problematic patches before shipping them in a stable release. 

Among the other problems that Greg is trying to address is maintainers who never mark any patches for the stable tree. "We need to fix that," he said. There is also the problem Dave mentioned, where stable fixes live in linux-next for [![\[Greg Kroah-Hartman\]](https://static.lwn.net/images/conf/2013/lce-ks/GregKH-sm.jpg)](<https://lwn.net/Articles/572018/>) months; "don't do that." If a patch is tagged for stable, he said, that means it should go out soon, not languish for months. James said that sometimes he will hold stable fixes because he needs people to test them; he does not have every piece of SCSI hardware, and so cannot verify that every patch works as advertised. 

Greg went on to say that it would be nice to have some way to automate the task of figuring out how far back a given patch needs to be backported. To that end, some developers are proposing the addition of a "Fixes:" tag to bug-fix patches. That tag would include the SHA hash of the commit that caused the original problem, along with that patch's subject line. Including the hash of the bad commit is better than just putting an initial version; it helps maintainers of non-mainline trees figure out if the fix applies to their version of the kernel or not. 

Linus jumped in to say that he would like everybody to run this command in their repositories: 

```
git config core.abbrev 12
```

That causes git to abbreviate commit hashes to 12 characters. The default of seven characters is too small to prevent occasional hash collisions in the kernel; it was, he said, a big mistake made early in git's history. He also noted that he spends a lot of time fixing up hashes in patch, many of which are "clearly bogus." Most of the problems, most likely, are caused by the rebasing of trees. 

James asked: what should be done about patches that should have been marked for stable but, for whatever reason, did not get tagged? The answer was to send the relevant mainline git IDs to stable@vger.kernel.org; the rest will be taken care of. 

Arnd Bergmann asked whether there had been complaints about the dropping of support for the 3.0.x series. Greg answered that people are mostly happy and not complaining. One person is considering picking up 3.0 maintenance, but Greg does not think that will happen. 

Other problems that Greg mentioned including people reporting that things have been broken in stable, but the real problem is in the mainline. Those reports usually come, he said, from people who are not testing Linus's -rc releases. He also mentioned "certain distributors" who are not good about sending in the fixes they apply to their own kernels. Those fixes tend to be especially useful, since they were applied in response to a real problem that somebody encountered. If anybody wants to help out the stable process, he said, digging through distributor kernels for these fixes would be a useful thing to do. 

As the session wound to a close, Greg was asked about what he does with patches that do not apply to older kernels. His response is that he will bounce them back to the maintainer for backporting. Subsystem maintainers, he said, need not worry about patches being tweaked on their way into -stable. 

[Next: [Testing](<https://lwn.net/Articles/571991/>)].
