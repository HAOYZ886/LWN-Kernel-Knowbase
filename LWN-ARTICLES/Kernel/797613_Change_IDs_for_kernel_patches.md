---
title: Change IDs for kernel patches
url: https://lwn.net/Articles/797613/
date: "August 29, 2019"
category: "Development model-Patch management"
author: "By Jonathan Corbet August 29, 2019"
---

By **Jonathan Corbet**  
August 29, 2019

For all its faults, email has [long proved](<https://lwn.net/Articles/702177/>) to be an effective communication mechanism for kernel development. Similarly, Git is an effective tool for source-code management. But there is no real connection between the two, meaning that there is no straightforward way to connect a Git commit with the email discussions that led to its acceptance. Once a patch enters a repository, it transitions into a new form of existence and leaves its past life behind. Doug Anderson recently went to the ksummit-discuss list with [a proposal](<https://lwn.net/ml/ksummit-discuss/CAD=FV=UPjPpUyFTPjF-Ogzj_6LJLE4PTxMhCoCEDmH1LXSSmpQ@mail.gmail.com/>) to add Gerrit-style change IDs as a way of connecting the two lives of a kernel patch; the end result may not be quite what he was asking for. 

The [Gerrit](<https://www.gerritcodereview.com/>) code-review system needs to be able to track multiple versions of the same patch; to do so, it adds a Change-Id tag to the patches themselves: 

```
Change-Id: I6a007dfe91ee1077a437963cf26d91370fdd9556
```

The tag is automatically added to the first version of a new patch; developers are expected to retain that tag when posting subsequent versions so that Gerrit can associate the new and old versions. These tags are useful for Gerrit, but they have never been welcome in the kernel community; Anderson posted his missive in the hope of changing that attitude and getting the community to allow (or actively encourage) the use of change IDs in patches: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The basic summary is that I'd like there to be some way to track a logical patch over its lifetime. I don't believe there is a reliable (non-heuristic) way to do this today and I think Change-Id provides a nice solution. While we could come up with a new and different solution (because Change-Id was not invented here), it feels like adopting Change-Id is convenient and easy and provides a true benefit. 

The problem Anderson describes is real enough; your editor, who spends a lot of time digging up old versions of patch postings to work out how a patch has evolved over time, can attest to that. Guenter Roeck [complained](<https://lwn.net/ml/ksummit-discuss/dc1a4c98-5e29-094c-ead8-889df35de811@roeck-us.net/>) that he has to ""use a combination of subject analysis and patch content analysis using fuzzy text / string comparison, combined with an analysis of the patch description"" to determine whether a given patch has been merged. There seems to be little doubt that the community as a whole would appreciate a better way to associate a patch's history over time and its final resting place in the kernel repository. That is about where the agreement stops, though. 

Linus Torvalds was quick to [reject](<https://lwn.net/ml/ksummit-discuss/CAHk-=whFbgy4RXG11c_=S7O-248oWmwB_aZOcWzWMVh3w7=RCw@mail.gmail.com/>) the idea of putting a bare change ID into patch changelogs, citing the same reasoning that has kept those IDs out thus far: they are really only useful to whoever put that ID into the changelog in the first place. Gerrit change IDs are useful to people who know which Gerrit instance is tracking the patch in question and who actually have access to that instance. For everybody else, it's just a number that is just extra noise in the changelog; as Torvalds put it: ""A 'change ID' that I can't use to look anything up with is completely pointless and should be removed from kernel history"". That assertion also implies, of course, that an ID that _can_ be looked up by third parties might have some value. 

One way to make a Gerrit change ID useful, he suggested, would be to turn it into a publicly accessible web link; then anybody could follow the link, see whatever other information exists, and track the history of the patch. Olof Johansson [disliked](<https://lwn.net/ml/ksummit-discuss/CAOesGMgryBghLuTSufjhOUFUFDvL2Jv7qJ3uwaZUwCkMXpEmng@mail.gmail.com/>) that idea, saying that the Gerrit server could be shut down, making the link useless. Ted Ts'o [responded](<https://lwn.net/ml/ksummit-discuss/20190823013619.GA8130@mit.edu/>) that such a fate could befall any web link, including others (such as bugzilla links) that are accepted in changelogs now. 

There may be other ways to solve this problem, though. The idea that Torvalds [liked the best](<https://lwn.net/ml/ksummit-discuss/CAHk-=whxp0Vm5V6MtaRNC0_ou0=U4-+Y7Ktzq6osU8JgHy_xhQ@mail.gmail.com/>) — and which seems to have the widest support across the community — is to use the unique ID that is already associated with a patch posting, which is the message ID created by the poster's email client: 

The first time it gets magically and reliably created for you without you having to do a single thing. The second time, you just look it up. 

Ta-daa - you have a "uuid" that is useful to others, and that describes the whole series unambiguously. 

There are a few ways this ID could be presented, but the most popular way is to create a "Link:" tag containing a link to the posting of the patch in a public mailing-list archive server (generally [lore.kernel.org](<https://lore.kernel.org/lists.html>) in recent times). This is not a new practice; it appears to have first been used for [this patch](<https://git.kernel.org/linus/f994d99cf140>) applied by H. Peter Anvin in 2011. Use of this tag is not universal, but it is growing; the number of patches in recent kernels carrying Link: tags is: 

> Release| Tags| Percent  
> ---|---|---  
> 4.18 | 1,413 | 10.6%  
> 4.19 | 1,944 | 13.8%  
> 4.20 | 1,609 | 11.6%  
> 5.0 | 1,778 | 13.9%  
> 5.1 | 1,908 | 14.6%  
> 5.2 | 2,295 | 16.4%  
> 5.3 | 2,614 | 18.4%  
  
Creation of this tag is relatively easy; it can be entirely automated at the point where a patch is applied to a Git repository. But it doesn't solve the entire problem; it can associate a commit with the final posting of a patch on a mailing list, but it cannot help to find previous versions of a patch. Generally, the discussion of the last version of a patch is boring since there is usually a consensus at that point that it should be applied. It's the discussion of the previous versions that will have caused changes to be made and which can explain some of the decisions that were made. But kernel developers are remarkably and inexplicably poor at placing the message ID of the final version of a patch into the previous versions. 

The most commonly suggested solution to that problem is not fully automatic. Developers like [Thomas Gleixner](<https://lwn.net/ml/ksummit-discuss/alpine.DEB.2.21.1908231740020.1896@nanos.tec.linutronix.de/>) and [Christian Brauner](<https://lwn.net/ml/ksummit-discuss/20190828122234.3femrepecvswo7ws@wittgenstein/>) argued in favor of adding a link to previous versions of a patch when posting an updated version. Gleixner called for a link to the cover letter of the prior version, while Brauner puts links to all previous versions. Either way, an interested developer can follow the links backward to see how a patch series has changed, along with the discussions that led to those changes. 

A convention like that would provide most or all of what developers like Anderson are asking for. It would, however, require that developers do some work to insert those links, and not everybody is convinced that this will ever happen. Dmitry Torokhov [said](<https://lwn.net/ml/ksummit-discuss/20190823161947.GA112509@dtor-ws/>) that he could not be bothered: 

As a patch submitter, I frankly could not care less about proper trace, history, etc. I might be putting cover letter and outline the version changes, but I am doing that to reduce friction and help committer to land my change. That's it. 

Developers, he said, simply would not do the extra work to add links to previous postings to their cover letters. Anderson also [asserted](<https://lwn.net/ml/ksummit-discuss/CAD=FV=VJt4+XRj59h6tmma5LfU52E_6QOSQVjh3T3M+1V=eMsQ@mail.gmail.com/>) that ""the adoption rate will be near to zero"". Such concerns have merit; it is hard to force a community of thousands of developers to do more work for every patch they submit. But without their cooperation, this idea will not go far. 

The answer, naturally, is to provide tools that make the right thing happen with a minimum of extra work. Gleixner [described](<https://lwn.net/ml/ksummit-discuss/alpine.DEB.2.21.1908232150580.1939@nanos.tec.linutronix.de/>) the setup he uses with [Quilt](<https://savannah.nongnu.org/projects/quilt>), but it seems unlikely that all developers will find it useful for their purposes. Joel Fernandes [described](<https://lwn.net/ml/ksummit-discuss/CAJWu+oq8jFnJ4iJ+R3swr-93eMwDgbWXe1W2Aiu-r+tuYSY5Ag@mail.gmail.com/>) a tool that he is considering writing that might be more generally useful. Greg Kroah-Hartman [described](<https://lwn.net/ml/ksummit-discuss/20190828090837.GA31704@kroah.com/>) it as overly complex, though, and suggested simply posting patches as a reply to previous versions, but others pointed out that not all mailers would make that entirely easy to do. 

Ts'o more-or-less ended the discussion by [saying](<https://lwn.net/ml/ksummit-discuss/20190828135820.GA24857@mit.edu/>) that it was time for interested developers to go off and implement a prototype of the tools they have in mind for this task. Then the code could be evaluated to see how it actually works. ""Trying to pick something before people who actually have to use it day to day have had a chance to try it in real life is how CIO's end up picking Lotus Notes"". That is where things stand now; the next step will come about when somebody comes forward with a tool that might provide a better solution to the problem. Until then, we'll have to continue to use fuzzy string comparisons and other tricks to track the history of patches in the repository.
