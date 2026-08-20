---
title: The many names of commit 55039832f98c
url: https://lwn.net/Articles/1005222/
date: "January 16, 2025"
category: "Development model-Stable tree"
author: "By Jonathan Corbet January 16, 2025"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 16, 2025

The kernel is, on its face, a single large development project, but internally it is better viewed as 100 or so semi-independent projects all crammed into one big tent. Within those projects, there is a fair amount of latitude about how changes are managed, and some subsystems are using that freedom in the search for more efficient ways of working. In the end, though, all of these sub-projects have to work together and interface with kernel-wide efforts, including the stable-release and CVE-assignment processes. For some time, there has been friction between the direct rendering (DRM, or graphics) subsystem and the stable maintainers; that friction recently burst into view in a way that shows some of the limitations of how the kernel community manages patches. 

One of the key rules for the stable releases is that every patch must first land in the mainline kernel before being considered for the stable trees. The rule exists to ensure that stable patches have had a bare minimum of wider exposure and testing, but also to ensure that no fixes fall through the cracks and miss the mainline entirely. Exceptions to this rule are rare, and usually involve urgent security fixes. In practice, this rule means that any patch proposed for the stable updates should include a line, in its changelog, providing the mainline commit ID for that patch. 

On January 10, [this patch to the Xe graphics driver](<https://lore.kernel.org/stable/20250110205341.199539-1-umesh.nerlige.ramappa@intel.com/>), written by Umesh Nerlige Ramappa, was sent to the mailing list dedicated to potential stable-update patches. It duly included this line in its changelog: 

```
commit 55039832f98c7e05f1cf9e0d8c12b2490abd0f16 upstream
```

The only problem is that the commit named there does not, as of this writing, exist in the mainline kernel repository. It _does_ exist in linux-next, and can be expected to land in the mainline during the 6.14 merge window. So this would appear to be a violation of the mainline-first rule — this fix is being proposed for the stable updates before it hits the mainline repository. So something strange is happening here. There is a clue to be found in another line in the patch as posted to the stable mailing list — but not in the version that appears in linux-next: 

```
(cherry picked from commit 55039832f98c7e05f1cf9e0d8c12b2490abd0f16)
```

This line can also be found in commit `f0ed39830e60`, which contains the same fix and [_is_ in the mainline](<https://git.kernel.org/linus/f0ed39830e60>); it landed there in time for the [6.13-rc6](<https://lwn.net/ml/all/CAHk-=wgjfaLyhU2L84XbkY+Jj47hryY_f1SBxmnnZi4QOJKGaw@mail.gmail.com/>) prepatch. So the change, as posted to the stable list, is indeed a legitimate stable candidate, but figuring that out takes some extra work, and the upstream citation contained in its changelog must either be corrected, or it will forever refer to a commit that did not, in truth, fix the bug in the mainline. 

This situation comes about as a result of the way the DRM subsystem organizes its maintainers. DRM is a large subsystem containing many drivers that are some of the largest components within the kernel. Just like maintaining the kernel in a single repository led to scalability problems many years ago, maintaining DRM that way would not work now. So the DRM subsystem is managed by way of a set of sub-subsystem trees that come together in the drm-next repository. A relatively large number of developers are empowered to commit to these repositories, with the result that the maintainership work is widely distributed. 

So far so good; the place where the friction comes in is the DRM community's wide use of cherry-picking to move individual commits between repositories as needed. The commit in question had been given ID `55039832f98c` on its way into the drm-next repository; that repository feeds into linux-next, so the commit appears there with that ID. At some point, it was identified as a fix that should go into 6.13 rather than waiting for the 6.14 merge window; that resulted in it being cherry-picked into a separate fixes branch as `f0ed39830e60`. The "cherry picked from" line was added automatically at that time. Later on, the commit was cherry-picked _again_ into another branch for submission to the stable process, resulting in another line in the changelog: 

```
(cherry picked from commit f0ed39830e6064d62f9c5393505677a26569bb56)
```

This is the form in which it found its way to the stable maintainers, one of whom, Greg Kroah-Hartman, [was not pleased](<https://lwn.net/ml/all/2025011215-agreeing-bonfire-97ae@gregkh>). This profusion of IDs for a single commit, and the multiple appearances of a commit in the repositories, makes it difficult for the stable maintainers to make decisions about which commits should be backported or have CVE numbers assigned to them. 

In response, Kroah-Hartman said: 

> I give up. You all need to stop it with the duplicated git commit ids all over the place. It's a major pain and hassle all the time and is something that NO OTHER subsystem does. 
> 
> Yes, I know that DRM is special and unique and running at a zillion times faster with more maintainers than any other subsystem and really, it's bigger than the rest of the kernel combined, but hey, we ALL are a common project here. If each different subsystem decided to have their own crazy workflows like this, we'd be in a world of hurt. 

Sometimes, the existence of multiple IDs can cause the fix for a bug to appear to be merged before the bug itself. For example, [commit `a6dd15981c03`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a6dd15981c03>), merged for 6.12-rc7, claims to be a fix for [commit `c9b7c809b89f`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c9b7c809b89f>), which landed in 6.13-rc1. Kernel development moves quickly, but one would be hard put to say that it moves _so_ quickly that, by way of relativistic effects, the causality between a bug and the patch that fixes it can appear to be violated. This sort of apparent inversion between cause and effect is not uncommon in the kernel's repository, and it can definitely lead to confusion. 

Most subsystems commit short-term fixes to a separate branch from the outset, then push that branch to Linus Torvalds occasionally. If needed, the maintainers of that subsystem may also merge the fixes branch into their "next release" branch. When this process is followed, the fix will retain the same commit ID throughout, and the kinds of problems described here will not arise. 

The DRM maintainers, though, have said clearly that they believe such a process cannot work at the scale seen in that subsystem. Dave Airlie [suggested](<https://lwn.net/ml/all/CAPM=9txn1x5A7xt+9YQ+nvLaQ3ycekC1Oj4J2PUpWCJwyQEL9w@mail.gmail.com>) fixing the tools used to maintain the stable releases instead. Simona Vetter [explained](<https://lwn.net/ml/dri-devel/Z4d6406b82Pu1PRV@phenom.ffwll.local/>) how things work from a graphic developer's point of view, pointing out that, for most of them, the mainline kernel is a distant downstream consumer of their work. The way a patch appears in drm-next, including its ID, tends to be more relevant; that is why a drm-next ID can be cited in a stable-candidate patch, even if that patch entered the mainline with a different ID. 

Vetter [said](<https://lwn.net/ml/all/Z4aJ9XU7y8-Sl36j@phenom.ffwll.local>) that most of the problems experienced by the stable maintainers can be solved by simply looking at all of the commit IDs found in the changelog, including the cherry-picked ones, when deciding whether a change has appeared in another tree (such as the mainline). Vetter [added](<https://lwn.net/ml/all/Z4eAKvX1A3-sV1XE@phenom.ffwll.local>) that: ""We won't change our process, because I couldn't find the maintainer volunteers to make that happen"", noting also: ""This shit is easy, except somehow here we are almost a decade later"". 

Sasha Levin, another stable-release maintainer, [pointed out](<https://lwn.net/ml/all/Z4ft1fFjbwy0EF-X@lappy>) that problems still remain, especially in cases where a commit ends up in the mainline more than once (which happens reasonably frequently in the DRM subsystem). ""So no, this isn't a simple trace-the-cherry-pick-tags exercise"". Vetter [disagreed](<https://lwn.net/ml/all/Z4gGKVFFXBQEm_OB@phenom.ffwll.local>), though, providing a detailed description of how the chain of cherry-picking develops and concluding ""sometimes you do have to do a bit more cherry-pick tracing than what you've done"". 

Perhaps what is really on display here is the limitations inherent in using a commit ID to identify a change; that ID is firmly tied to just how a commit lands in a specific branch. Some tools, such as [Gerrit](<https://www.gerritcodereview.com/>), place a separate "change ID" into each changelog to identify a change uniquely through both movements through repositories and revisions by the developer; the kernel community, though, has shown little interest in using change IDs and generally prohibits them from appearing in changelogs. 

Toward the end of the conversation, Vetter [posted](<https://lwn.net/ml/all/Z4jV_cPkg82X6FrY@phenom.ffwll.local>) ""the generic algorithm"" for locating commits in the various trees, acknowledging that: ""It's a bit madness, but more importantly, it's scriptable madness"". Kroah-Hartman [agreed](<https://lwn.net/ml/all/2025011610-request-unleveled-8191@gregkh>) that it is ""total madness"", saying that it takes far too long to execute, but also seemed resigned to dealing with it. ""I know DRM isn't the only offender here, many commits flow through multiple trees at the same time"". He said he would work on a solution during the coming merge window. With luck, this work will help to bring an end to this long-running disagreement and reduce the impedance mismatches between different practices across the kernel community.
