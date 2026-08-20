---
title: A discussion on stable kernel workflow issues
url: https://lwn.net/Articles/705220/
date: "November 1, 2016"
category: "Development model-Stable tree"
author: "By Jonathan Corbet November 1, 2016 2016 Kernel Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 1, 2016

* * *

[2016 Kernel Summit](<https://lwn.net/Articles/KernelSummit2016/>)

The opening session at the 2016 Kernel Summit, led by Jiri Kosina, had to do with the process of creating stable kernel updates. There is, he said, a bit of a disconnect between what the various parties involved want, and that has led to trouble for the consumers of the stable kernel releases. 

Jiri's point of view was centered on his role as a distribution kernel maintainer. Consumers like him want a number of things from the stable kernel releases, including fixes for user-visible functional problems, fixes for bugs that crash the system, and fixes for severe performance regressions. What they do _not_ want are new features or minor performance improvements; the latter have often been shown to regress performance for other workloads. 

Perhaps the biggest thing in the "don't want" column, though, is something that has caused quite a bit of trouble in the past: fixes for bugs that are not actually present. There have been a number of cases of bogus "fixes" that have broken things, causing big headaches for distributors, who must spend a lot of time figuring out what has gone wrong. Just because a patch applies cleanly to an older kernel does not mean that it actually belongs there, but that distinction often seems to get lost. 

Part of that, perhaps, is a result of what the _producers_ of stable kernels want: a process that scales. Stable releases are done by a small group of developers; they don't have a lot of time to spend on each proposed fix. They want to include all of the fixes that make sense, but depend on others to tell them when fixes actually do make sense. 

From his point of view as a distribution maintainer, Jiri said that one of the biggest issues with the stable process is that it is not clear why specific patches got into stable. The "CC: stable" tag applied to patches is an opt-in mechanism, and the decision is often made by people who are neither the author of the patch nor the maintainer of the relevant subsystem. The review process is far too lenient; it is a passive approval approach that lets stuff get in unless somebody goes out of their way to block it. Maintainers often respond to proposed stable inclusions with the [![\[Jiri Kosina\]](https://static.lwn.net/images/conf/2016/ks/JiriKosina-sm.jpg)](<https://lwn.net/Articles/705227/>) equivalent of "oh yeah, whatever" without really thinking about the problem. The review barrier is too low; somebody needs to be thinking about the semantics of every stable patch. 

James Bottomley noted that the typical maintainer response is to think that they already reviewed a specific patch when they first accepted it; they aren't sure what else they should do when a patch comes around in a stable release. Mark Brown added that maintainers often can't remember what a particular ancient kernel looked like, so it is hard for them to say whether a specific patch makes sense there or not. 

Christoph Hellwig pointed out that not all stable kernels are equal. The current review process is reasonable for recent kernels, he said, but it does not work as well for the long-term stable releases. Stable kernel maintainer Greg Kroah-Hartman asked what "good" meant in general. There are always going to be bugs, what is the acceptable rate when five or six patches are being applied to a two-year-old kernel every day? 

Jiri said that there is always room for improvement. One possible way to make things better might be to require a `Fixes:` tag for every patch going into a stable kernel release. That tag identifies the bug that a patch is meant to fix; without it, the stable maintainers don't know if a patch fixes a bug that is actually present in an older kernel. An alternative might be a new tag specifically related to inclusion into a stable kernel; it would mean "I have thought about this responsibly." That tag should specify the version(s) of the kernel it should be applied to. There could also be a tag for patches that should _not_ be considered for stable releases. 

Ben Herrenschmidt asked about whether the stable kernels should include patches to make new hardware work. Jiri responded that those patches aren't really wanted, but he didn't care too much if they don't break things. Greg added that addition of new device IDs and quirks is pretty common in the stable kernels. Laura Abbot noted that there is often a fine line between a "feature" and a "bug fix." The direct rendering subsystem, for example, often comes up with large changes to make specific hardware features perform well; without them, things run slowly. DRM subsystem bug-fix patches can also be quite large. 

Rafael Wysocki said that, often, a maintainer knows that things broke, but is not sure of which change caused the problem. Jiri responded that, in that case, the fix should not go into the stable releases. Ben wondered how a problem could be fixed if the developer didn't know what caused the problem. Linus chimed in to note that requiring a `Fixes:` tag could lead to the addition of bogus tags when the maintainer doesn't know. Maintainers should not do that, he said, but Mark worried that it could happen as a "hoop-jumping" exercise. 

Jiri said that, one way or another, somebody has to go through the exercise of thinking about which stable kernels a patch should be applied to. The result can take the form of either a `Fixes:` tag or a list of relevant versions, as long as that work has been done. 

Dan Williams suggested that developers should add more test cases to demonstrate bugs and prove their fixes. These tests could be run against older kernels to show whether they need the fix — and whether the fix works as expected. Steve Rostedt said he will often add tests in response to bugs in his code; he suggested the addition of a new tag listing a test to run for a patch. But Linus was against the addition of new tags, especially for passing information to other humans. That sort of thing should just be explained in the commit message, he said. If we have too many tags, their usage will be inconsistent at best. 

Linus went on to point out a specific case of things going wrong. The stable trees recently shipped [a three-line patch](<https://git.kernel.org/linus/1c109fabbd51863475cd12ac206bdd249aee35af>) that made a trivial root exploit possible on older CPUs. He had suggested removing the patch from the stable series, but the stable maintainers all chose to apply a 100-line fix instead. That was, he said, the wrong decision. If a stable tree takes a patch that breaks things, the response should not be to add more patches. Greg said that the stable maintainers would, as a general rule, rather have the same changes that the mainline has so that the bugs are all the same. 

Ted Ts'o said that there are a lot of device vendors out there who are not actually using the stable releases; instead, they are cherry-picking patches that look relevant. If, in this case, they got the problematic three-line patch but missed the subsequent fix, the results would be bad. In general, he said, the group had not yet talked much about how the stable kernels are actually being used. Rafael said that broken patches should simply be reverted from the stable kernels rather than fixed further; that way, it will be clear to others where the broken patch was. But Ben protested that patches often come in a series; reverting just one can break things further. 

Andy Lutomirski said that the fixup patch in this case was partly his work, even though it didn't have his name on it when it was applied. That fix should never have been applied to the stable kernels, he said; stable maintainers, in general, should never apply complex changes without asking first. 

James said that the failure this time around was in the review process, but Linus replied that the patch was, in fact, "fine and correct." The problem is that it [depended on another change](<https://lwn.net/Articles/705226/>) that went into the 4.6 kernel. He doesn't blame that specific patch for the trouble; the bug was not obvious, and the backport was clean. It would have taken a "superhuman" to realize that the patch would be problematic in 4.1 without the exception-handling change that was applied in 4.6. The failure was when the patch was revealed to have broken things; at that point, it should have been reverted immediately. Mistakes will happen and the stable kernels will not be perfect; but, he said, those kernels are too eager to accept patches, and to accept more rather than reverting a patch that went wrong. 

As the discussion wound down, perhaps the one solid conclusion was that problematic patches should indeed be reverted rather than fixed; that policy was immediately applied in subsequent stable kernel releases ([example](<https://lwn.net/Articles/705221/>)). There will likely be more pressure for each patch destined for the stable releases to carry a `Fixes:` tag or some version information making it clear that the relevant bug is present. And, hopefully, stable releases will be a little more stable in the future.
