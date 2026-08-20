---
title: Code of (still) uncertain origin
url: https://lwn.net/Articles/195643/
date: "August 15, 2006"
category: Copyright issues
author: 
---

In [last week's episode](<http://lwn.net/Articles/194729/>), we looked at the story of the new Thinkpad embedded controller driver and its author "Shem Multinymous." The situation had been put on hold after Pavel Machek had offered to sign off on the code, and the discussion died down for a bit. Not for long, though. 

Robert Love, the author of the accelerometer driver which (among other things) is replaced by this code, [reviewed it](<https://lwn.net/Articles/195648/>), noting ""I am glad someone has apparently better access to hardware specs than I did"" That [brought Andrew Morton back in](<https://lwn.net/Articles/195649/>), saying: 

This situation is still a concern. From where did this additional register information come? [...] 

We're setting precedent here and we need Linus around to resolve this. Perhaps we can ask "Shem" to reveal his true identity to Linus (and maybe me) privately and then we proceed on that basis. The rule could be "each of the Signed-off-by:ers should know the identity of the others". 

That [is not good enough](<https://lwn.net/Articles/195652/>) for Greg Kroah-Hartman, however: 

For what it's worth, I'm not going to be handling these patches at all (normally the hwmon patches go to Linus through Jean and then through me.) If the original developer does not want to work in the open like the rest of us, I can respect that, but unfortunately I can't accept the risk of accepting their code. 

Jean Delvare has also [declined to look at the code](<https://lwn.net/Articles/195653/>), saying that the legal uncertainty is too strong. Shem Multinymous, on the other hand, seems willing to come clean to Linus and Andrew if that is what it takes to get the code into the kernel. So it is conceivable that things could happen that way, with the code bypassing the maintainers who would normally handle (and review) it. Some residual concern could remain, however, perhaps to the point that distributors would consider removing the code from the kernels they ship. 

"Shem" has also posted [two](<https://lwn.net/Articles/195654/>) [separate](<https://lwn.net/Articles/195655/>) messages on the provenance of the information used in this driver. The story, it seems, starts with a reverse-engineered Windows driver. Then, a real spec for the embedded controller chip was found. After that, it was mostly a matter of putting the pieces together. Or so it is said. 

If this story holds together, then the new code probably is something which can be merged into the mainline without worry; it should be at least as legitimate as the original driver which it replaces. But, even if it gets in, this code will have set a precedent of sorts: anonymous submissions (at least, those submitted under an obvious pseudonym) are going to have a hard time getting through the process. Nobody wants to be the person who guided bad code into the kernel.
