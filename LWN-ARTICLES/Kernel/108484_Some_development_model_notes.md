---
title: Some development model notes
url: https://lwn.net/Articles/108484/
date: "October 27, 2004"
category: Development model
author: 
---

There has been an increase in complaints about the 2.6 development model recently. Some observers are dismayed by the continued high rate of change in 2.6, and have posted calls for the creation of a 2.7 branch and restricting 2.6 to critical bug fixes only. Failure to separate development and maintenance in this way, it is said, hurts the reputation of the Linux kernel and subjects users to needless regressions. 

The interesting thing with this discussion is that the people objecting to the current development mode have not been able to point to much in the way of specific problems that have resulted from it. A few specific bugs have been listed, but most of those have been around for some time and cannot be said to have resulted from recently-merged new features. The only complaint which holds water might be [this one](<https://lwn.net/Articles/108507/>) regarding the plight of some out-of-tree kernel development project (PaX in particular). PaX, it seems, is stuck at 2.6.7 because its developers have not yet been able to respond to subsequent changes in internal interfaces. 

This argument, of course, does not get very far with most kernel developers. There is an increasing amount of pressure to get out-of-tree projects to submit their code and become part of the mainline tree. Code which is in the mainline gets fixed (usually) when internal interfaces change, but only the original developers can maintain external code. So the standard answer to this sort of complaint is "merge your patches." Changes in the development model to accommodate out-of-tree projects are unlikely. 

Not every 2.6 kernel release has been 100% stable, but the same can be said of previous stable kernel series as well. What _is_ different this time is that 2.6 has a great many new features and improvements which would not have been merged under the older model. Many of those improvements would, instead, have been backported by distributors and shipped as part of the "stable" kernel anyway. Under the new scheme, those patches are merged into the mainline, are debugged by everybody involved, and are available to all users. It seems unlikely that most users truly wish to go back to the old days, when distributors shipped highly divergent kernels with (literally) hundreds of patches. 

There are occasional requests for bugfix-only "point" releases for the major 2.6 kernels. Rather than wait for 2.6.10, and take all of the other changes which come with that kernel, some people wish for a 2.6.9.1 (and so on) with just the important fixes. The standard response to _that_ request is that anybody can create and maintain such a tree. So far, however, there has not been sufficient demand for this tree to motivate somebody to actually do the work. (It should be noted, though, that Alan Cox has restarted posting his "-ac" patches, which contain fixes that are, in his opinion, important). 

All of the above distracts from the _real_ development model discussion: what Linus should call his intermediate releases. There is a steady stream of objections to the "-rc" scheme, since, in fact, very few such kernels are actually release candidates. Linus pondered the issue, but decided to call the first 2.6.10 prepatch [2.6.10-rc1](<https://lwn.net/Articles/107822/>) anyway: 

And the fact is, I can't see the point. I'll just call it all "-rcX", because I (very obviously) have no clue where the cut-over-point from "pre" to "rc" is, or (even more painfully obviously) where it will become the final next release. So to not overtax my poor brain, I'll just call them all -rc releases, and hope that developers see them as a sign that there's been stuff merged, and we should start calming down and seeing to the merged patches being stable soon enough. 

So the -rc names will continue for the foreseeable future.
