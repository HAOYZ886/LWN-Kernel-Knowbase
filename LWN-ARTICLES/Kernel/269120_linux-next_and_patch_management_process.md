---
title: "linux-next and patch management process"
url: https://lwn.net/Articles/269120/
date: "February 13, 2008"
category: "Development model-linux-next; linux-next"
author: "By Jonathan Corbet February 13, 2008"
---

By **Jonathan Corbet**  
February 13, 2008

The kernel development process operates at a furious pace, merging on the order of 10,000 changesets over the course of a 2-3 month release cycle. There have been many changes over the last few years which have helped to make this level of patch flow possible, and the process has been optimized significantly. An ongoing discussion on the kernel mailing list has made it clear, though, that a truly optimal solution has not yet been found. 

It started with [the announcement of the linux-next tree](<http://lwn.net/Articles/268881/>). This tree, to be maintained by Stephen Rothwell, is intended to be a gathering point for the patches which are planned to be merged in the next development cycle. So, since we are currently in the 2.6.25 cycle, linux-next will accumulate patches for 2.6.26. The idea is to solve the patch integration issues there and reduce the demands on Andrew Morton's time. 

The question which was immediately raised was this: how do we deal with big API changes which require changes in multiple subsystems? These changes are already problematic, often requiring maintainers to rework their trees in the middle of the merge window. Trying to integrate such changes earlier, in a separate tree, could bring a new set of problems. There will be a lot of conflicts between patches done before and after the API change, and somebody is going to have to put the pieces back together again. Andrew does some of that now, but the problem is big enough that not even Andrew can solve it all the time. The bidirectional SCSI patches merged for 2.6.25 were [held up as an example](<https://lwn.net/Articles/269175/>); that change required coordinated SCSI and block layer patches, and it never was possible to get the whole thing working in -mm. 

Arjan van de Ven [asserted](<https://lwn.net/Articles/269182/>) that the only way to make large API changes work is to merge them first, at the beginning of the merge window. The merged patch would fix all in-tree users of the changed API, as is the usual rule. Maintainers of all other trees could then merge with the updated mainline, fixing any new code which might be affected by the API change. This is, essentially, the approach which was taken for the big device model changes in 2.6.25; they hit the mainline at the beginning of the merge window, then everybody else got to adapt to the new way of doing things. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Greg Kroah-Hartman [worries that this approach is not sufficient](<https://lwn.net/Articles/269187/>), especially when live trees are being merged. If an API change in one tree forces a change to a separate tree, the coordination issues just get hard. Keeping the secondary changes in the primary tree risks conflicts with patches in the proper subsystem tree. Patches which reach across trees are also, increasingly, being discouraged as making life harder for everybody. But the fixup patch will not apply to its nominal subsystem tree as long as the API change itself is not there. In the -mm tree, this sort of problem is glued together by a series of fixup patches maintained by Andrew; Greg says that the linux-next tree would need something similar. 

David Miller's [suggestion](<https://lwn.net/Articles/269201/>) was to resolve this sort of conflict through frequent rebasing of the -next tree. Rebasing is an operation (supported by git and other code management tools) which takes a set of patches against one tree and does what's required to make them apply to a different version of the tree. It can be quite useful for maintaining patches against a moving target - which kernel trees tend to be. David talked about how he rebases his (networking subsystem) trees frequently as a way of eliminating conflicts with the mainline and, in the process, cleaning some cruft out of the development history. 

It turns out, though, that this frequent rebasing is not popular with the developers who are downstream of David. Rebasing the tree forces all downstream contributors to do the same thing, and to deal with any merge conflicts that result. It makes it much harder to prepare trees which can be pulled upstream and creates extra work. 

This was where Linus [jumped into the conversation](<https://lwn.net/Articles/269208/>) and expressed his dislike of rebasing. He echoed the complaints from downstream developers that a constantly-rebased tree is hard to prepare patches against. It also confuses the development history, making changes to other developers' patches in silent ways. After somebody's patch set has been rebased, it is no longer the patches that were sent. So, [says Linus](<https://lwn.net/Articles/269210/>): 

So there's a real reason why we strive to *not* rewrite history. Rewriting history silently turns tested code into totally untested code, with absolutely no indication left to say that it now is untested. 

It is about here that Andrew Morton [commented](<https://lwn.net/Articles/269212/>) that git does not appear to be matching entirely well with the way that kernel developers work. Some of the solution may be found in tools more oriented toward the management of patch queues - such as quilt. There may be a renewed push to get more quilt-like functionality built into git (along the lines of the [stacked git](<http://procode.org/stgit/>) project) in the near future. 

Linus is also [not entirely pleased](<https://lwn.net/Articles/269214/>) with how the integration of patches only happens in the mainline: 

I'm also a bit unhappy about the fact you think all merging has to go through my tree and has to be visible during the two-week merge period. Quite frankly, I think that you guys could - and should - just try to sort API changes out more actively against each other, and if you can't, then that's a problem too. 

His suggestion is that a separate git tree should be created to contain a large API change - and nothing else. Affected subsystem maintainers could then merge that tree and develop against the result. In the end, all of the pieces should merge nicely in the mainline. 

This approach raises a number of interesting issues. The API-change tree has to be agreed upon by everybody, and it must be quite stable - lots of changes at that level will create downstream trouble. There must also be a high degree of confidence that this API-change tree will, in fact, get merged into the mainline; should Linus balk, everybody else's trees will no longer be applicable to the mainline. Replacing the current "tree of trees" patch flow with something messier could create a number of coordination issues. And there are fears that a mainline tree built from this process would fail to build in many of its intermediate states, which would make tools like "git bisect" much harder to use. Even so, it could be part of the long-term solution. 

Linus also took the opportunity to complain about large-scale API changes in general: 

Really. I do agree that we need to fix up bad designs, but I disagree violently with the notion that this should be seen as some ongoing thing. The API churn should absolutely *not* be seen as a constant pain, and if it is (and it clearly is) then I think the people involved should start off not by asking "how can we synchronize", but looking a bit deeper and saying "what are we doing wrong?" 

He also [stated](<https://lwn.net/Articles/269219/>) that the costs of big API changes are high enough that we should, more often, stay with older interfaces, even if they are not as good as they could be. Others [disagreed](<https://lwn.net/Articles/269220/>), claiming that Linux must continue to evolve if it is to stay alive and relevant. 

The rate of change seems unlikely to fall in the near future. There may be some changes to how big changes are done, though. As [suggested](<https://lwn.net/Articles/269222/>) by Ted Ts'o, more changes could be done by creating entirely new interfaces rather than breaking old ones. With Ted's scheme, the old interface would be marked "deprecated" at the beginning of the merge window. Developers would then have the entire development cycle to adjust to the change, and the deprecated interface would be removed before the final release. 

There is resistance to this approach, based on the observation that getting rid of deprecated interfaces tends to be harder than one would expect. But, still, it is a relatively painless way of making changes. The current transition (in the memory management area) from the `nopage()` VMA operation to `fault()` is an example of how it can work. Nick Piggin has been slowly changing in-tree users with the eventual goal of removing `nopage()` altogether. For now, though, both interfaces coexist in the tree and nothing has been broken. 

Like the kernel itself, its development process is undergoing constant change and (hopefully) improvement. As the development community and the rate of change continues to grow, the process will have to adjust accordingly. What changes come out of this discussion remain to be seen. But it's worth noting that Andrew Morton [fears](<https://lwn.net/Articles/269225/>) that the biggest problem - regressions and bugs - will be relatively unaffected.
