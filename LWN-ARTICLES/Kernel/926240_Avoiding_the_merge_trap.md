---
title: Avoiding the merge trap
url: https://lwn.net/Articles/926240/
date: "April 17, 2023"
category: "Development tools-Git"
author: "By Jonathan Corbet April 17, 2023"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 17, 2023

The kernel subsystem maintainers out there probably have a deep understanding of the sinking feeling that results from opening one's inbox and seeing a response from Linus Torvalds to a pull request. When all goes well, pull requests are acted upon silently; a response usually means that all has _not_ gone well. Several maintainers got to experience that feeling during the 6.3 merge window, which seemed to generate more than the usual number of grumpy responses related to merge commits. Avoiding that situation is not hard, though, with a bit of attention paid to how merges are done. 

When using a distributed system like Git, development is done in numerous parallel tracks, each of which has its own starting point. Even if a particular project starts at the tip of the mainline tree, the mainline itself is almost certain to have moved on by the time that work is ready to land there. Bringing independent lines of development back together is called "merging"; depending on what has changed, any given merge can be simple or a nasty mess of conflicting changes. 

There are many projects out there that disallow merges entirely, insisting that their development repository consist of a single sequence of commits. In such projects, developers must rebase their work before proposing it for upstream. The kernel project, though, has no problem with merges; indeed, the development process would not work without them. Consider that Torvalds did 208 pulls during the 6.3 merge window, each of which added commits to the mainline. If each pull request had to be rebased each time Torvalds did a pull to avoid a merge, little actual work would get done. Instead, almost every pull into the mainline results in another merge. 

Subsystem maintainers often do merges of their own, creating merge commits that end up being pushed into the mainline with the rest of their work. That is all normal, but it is those merge commits that tend to land maintainers in trouble. Why is it that Torvalds can create hundreds of merges during a typical development cycle, but subsystem maintainers get grumbled at? 

There are two things to watch out for when creating a merge in a subsystem tree. The first is that Torvalds insists that each merge be properly explained. Merges are commits too, and their changelog should say why the merge is being done. It is easy to just accept the default message that Git creates when adding a merge commit, but the resulting commit will have no useful explanation, which is the equivalent of waving a large red flag for Torvalds to see. Unexplained merges thus have a high likelihood of generating one of those unwanted replies. 

The other hazard is related, but arguably more subtle. Torvalds insists on an explanation of each merge because, it seems, he feels that many merges done by subsystem maintainers are unnecessary. "Back merges", where the current state of the mainline is merged back into a subsystem tree, come under extra scrutiny. Such merges clutter the development history and should be avoided if they are not needed. One way to determine whether a merge is needed is to explain the need for it; if the maintainer cannot write a changelog making the need for the merge clear, then perhaps the merge should not be done at all. 

Merges into subsystem trees are done for a number of reasons. These can include merging a topic branch that is ready to head upstream or to bring in work from another developer or maintainer. The need for the merge is clear in this case, and the form that the changelog describing it should take is also clear. Most of the merges done by Torvalds himself are of this type, and each one carries a changelog describing the new functionality that the merge brings. Your editor, who has made a habit of following traffic into the mainline for rather too many years at this point, can attest that the quality of those merge messages has increased considerably over the years. 

Almost any other merge is meant to bring in changes that take a different path into the mainline; back-merging from the mainline itself is the clearest example of that. Maintainers often manage two branches for work heading upstream, one for urgent fixes and one for larger work waiting for the next merge window. It is quite common to see, at some point, the fixes branch merged into the development branch, even though the fixes have likely already been sent to Torvalds separately. This is the kind of merge that Torvalds tends to question, though. 

[This email](<https://lwn.net/ml/linux-kernel/CAHk-=wh-u8dKyLtcDo4Vd=p==9hOH5D40de3tpC_rr_8eo9Lwg@mail.gmail.com/>) describes some of his thinking with regard to this sort of internal merge. One should not, he said, merge another branch just because it seems that some other work is going to need the changes found there. Instead, that other work should just be based on the fixes in the first place: 

> Because the "nice git way" to do that kind of thing is to actually realize "oh, I'm starting new work that depends on the fixes I already sent upstream, so I should just make a new topic branch and start at that point that I needed". 
> 
> And then - once you've done all the "new work" that depended on that state, only at *THAT* point do you merge the topic branch. 
> 
> And look - you have exactly the same commits: you have one (or more) normal commits that implement the new feature, and you have one merge commit, but notice how much easier it is to write the explanation for the merge when you do it *after* the work. 

As is so often the case, there is no hard rule here. If nothing else, it is not uncommon for new work to depend on both the fixes and supporting work that is in the development branch; that makes it hard to create a base for that work without merging first. And Torvalds acknowledged that some ""superfluous merges"" are not really a problem — as long as they are adequately explained. 

So, for subsystem maintainers who would prefer _not_ to get email from Torvalds in response to a pull request, there are a couple of simple rules to avoid the merge trap. Take the time to write an actual commit message explaining why the merge was done. But, before that, take a moment to think about whether there actually _is_ a reason to do the merge. The result should be a welcome reduction of merge-window stress and a cleaner commit history for the kernel as a whole.
