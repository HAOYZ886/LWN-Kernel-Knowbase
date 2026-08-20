---
title: "What's missing from our changelogs"
url: https://lwn.net/Articles/560392/
date: "July 24, 2013"
category: "Changelogs; Development model-Code review"
author: "By Jonathan Corbet July 24, 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 24, 2013

Tens of thousands of changes make their way into the mainline kernel every year. For most of those changes, the original motivation for the work is quickly forgotten; all that remains is the code itself and the changelog that goes with it. For this reason, kernel maintainers tend to insist on high-quality changelogs; as Linus recently [put it](<https://lwn.net/Articles/560393/>), ""We have a policy of good commit messages in the kernel."" Andrew Morton also famously pushes developers to document the reasons explaining why a patch was written, including the user-visible effects of any bugs fixed. Kernel developers do not like having to reverse engineer the intent of a patch years after the fact. 

With that context in mind, and having just worked through another merge window's worth of patches, your editor started wondering if our changelogs were always as good as they should be. A bit of scripting later, a picture of sorts has emerged; as one might expect, the results were not always entirely encouraging. 

#### Changelogs

A patch's changelog is divided into three parts: a one-line summary, a detailed change explanation, and a tags section. For the most trivial patches, the one-line summary might suffice; there is not much to add to "add missing include of foo.h", for example. For anything else, one would expect a bit more text describing what is going on. So patches with empty explanation sections should be relatively rare. 

As of this writing, just under 70,000 non-merge changesets have been pulled into the mainline repository since the release of the 3.5 kernel on July 21, 2012. Of those, 6,306 had empty explanations — 9% of the total. Many of them were as trivial as one might expect, but others were rather less so. 

Some developers are rather more laconic than others. In the period since 3.5, the developers most inclined to omit explanations were: 

> Developer| Count  
> ---|---  
> Al Viro| 570  
> Ben Skeggs| 224  
> Mark Brown| 213  
> Hans Verkuil| 204  
> Andreas Gruenbacher| 143  
> Axel Lin| 130  
> Philipp Reisner| 126  
> Antti Palosaari| 118  
> James Smart| 107  
> Alex Deucher| 85  
> Laurent Pinchart| 84  
> Kuninori Morimoto| 75  
> Eric W. Biederman| 75  
> Pavel Shilovsky| 72  
> Rafał Miłecki| 72  
> David S. Miller| 65  
> David Howells| 61  
> Peter Meerwald| 61  
> Maxime Ripard| 55  
> YOSHIFUJI Hideaki| 51  
  
For the curious, [a page listing the no-explanation patches merged by the above developers](<https://lwn.net/Articles/560397/>) is available. A quick look shows that a lot of patches with empty explanations find their way into the core virtual filesystem layer; many of the rest affect graphics drivers, audio drivers, Video4Linux drivers, and the DRBD subsystem. But they can be found anywhere; of the 1,065 changes that touched the `mm/` subdirectory, 46 lacked an explanation, for example. 

If one believes that there should be fewer patches with empty explanations going into the kernel, one might be inclined to push subsystem maintainers to be a bit more demanding in this regard. But, interestingly, it has become much harder to determine which maintainers have had a hand in directing patches into the kernel. 

#### Signoffs

The Signed-off-by line in the tags section is meant to document the provenance of patches headed into the mainline. When a developer submits a patch, the changelog should contain a signoff certifying that the patch can properly be contributed to the kernel under a GPL-compatible license. Additionally, maintainers who accept patches add their own signoffs documenting that they handled the patch and that they believe it is appropriate for submission to the mainline. In theory, by following the sequence of Signed-off-by lines, it is possible to determine the path that any change followed to get to Linus's tree. 

The truth is a little bit more complicated than that. To begin with, of the changes merged since 3.5, 79 had no signoffs at all. Roughly half of those were commits by Linus changing the version number; he does not apply a signoff to such changes, even for [those that contain added data](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=ad81f0545ef01ea651886dddac4bef6cec930092>) beyond the version number update. The rest are all almost certainly mistakes; a handful are the result of obvious formatting errors. See [the full list](<https://lwn.net/Articles/560401/>) for details. The mistakes are innocent, but they do show a failure of a process which is supposed to disallow patches that have not been signed off by their authors. 

Arguably, there is another class of patches that is more interesting: those that contain a single Signed-off-by line. Such patches have, in theory, been managed by a single developer who wrote the patch and got it into the mainline unassisted. One might think that only Linus is in a position to do any such thing; how could anybody else get a change into the mainline on their own? 

In fact, of the 70,000 patches pulled into the mainline during the period under discussion, 16,651 had a single signoff line. Of those, 11,527 (16% of the total) had no other tags, like Acked-by, Reviewed-by, or Tested-by, that would indicate attention from at least one other developer. For the purposes of this discussion, only the smaller set of patches has been considered. The most frequent committers of single-signoff patches are: 

> Developer| Count  
> ---|---  
> Al Viro| 891  
> Takashi Iwai| 525  
> Mark Brown| 492  
> Johannes Berg| 414  
> Alex Deucher| 391  
> Mauro Carvalho Chehab| 389  
> Ben Skeggs| 362  
> Greg Kroah-Hartman| 292  
> Trond Myklebust| 279  
> David S. Miller| 264  
> Felipe Balbi| 259  
> Tomi Valkeinen| 258  
> Arnaldo Carvalho de Melo| 172  
> Eric W. Biederman| 147  
> Josef Bacik| 145  
> Shawn Guo| 142  
> J. Bruce Fields| 141  
> Ralf Baechle| 132  
> Arnd Bergmann| 131  
> Samuel Ortiz| 129  
  
(See [this page](<https://lwn.net/Articles/560404/>) for a list of the single-signoff patches merged by the above developers). These results are, of course, a result of the use of git combined with the [no-rebasing rule](<https://lwn.net/Articles/328436/>). Once a patch has been committed to a public repository, it becomes immutable and can never again acquire tags like Signed-off-by. To pick one example from the list above, wireless developer Johannes Berg maintains his own tree for mac80211 changes; when he commits a patch, it will carry his signoff. Changes flow from that tree to John Linville's wireless tree, then to Dave Miller's networking tree, and finally to the mainline repository. Since each of those moves is done with a Git "pull" operation, no additional signoffs will be attached to any of those patches; they will arrive in the mainline with a single signoff. 

One might contend that patches become less subject to review once they enter the Git stream; they can be pulled from one repository to the next sight-unseen. Indeed, early in the BitKeeper era, developers worried that pull requests would be used to slip unreviewed patches into the mainline kernel. Single-signoff patches might be an indication that this is happening. And, indeed, important patches like [the addition of the `O_TMPFILE` option](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=60545d0d4610b02e55f65d141c95b18ccf855b6e>) went to the mainline with, as far as your editor can tell, no public posting or review (and no explanation in the changelog, for that matter). It also seems plausible that single-signoff patches merged into the sound subsystem or the Radeon driver (to name a couple of examples) have not been reviewed in detail by anybody other than the author; there just aren't that many people with the interest and skills to review that code. 

Without a chain of signoff lines, we lose more than a picture of which maintainers might have reviewed the patches; we also lose track of the path by which a patch finds its way into the mainline. A given changeset may pass through a number of repositories, but those passages leave no mark on the changeset itself. Sometimes that path can be worked out from the mainline repository history, but doing so can be harder than one might imagine, even in the absence of "fast-forward merges" and other actions that obscure that history. Given that the Signed-off-by line was introduced to document how patches get into the kernel, the loss of this information may be a reason for concern. 

The kernel community prides itself on its solid foundation of good procedures, including complete changelogs and a reliable signoff chain. Most of the time, that pride is entirely justified. But, perhaps, there might be room for improvement here and there — that is unsurprising when one considers that no project that merges 70,000 changes in a year can be expected to do a perfect job with every one of them. Where there is imperfection, there is room for improvement — though improving the signoff chain will be difficult as long as the tools do not allow it. But even a bit more verbiage in commit messages would be appreciated by those of us who read the patch stream.
