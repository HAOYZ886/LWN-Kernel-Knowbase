---
title: A policy for Link tags
url: https://lwn.net/Articles/1037069/
date: "September 11, 2025"
category: "Development model-Patch management"
author: "By Jonathan Corbet September 11, 2025"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 11, 2025

The Git source-code management system stores a lot of information about changes to code — but it does not hold everything that might be of interest to a developer who needs to investigate a specific change in the future. Commits in a repository are the end result of a (sometimes extended) discussion; often, that discussion will result in changes to the code that are not explained in the changelog. For some years now, many maintainers have followed the convention of applying a Link tag to commits that points back to the mailing-list posting of the change. Linus Torvalds has been expressing his dislike for this convention for a while, though, and its time appears to be coming to an end. 

Certain source-code management systems are able to track a change through multiple versions by assigning a "change ID" to the work. Git does not do that, though, so the kernel community does not have an easy way to look at the history of a patch. In a discussion prior to the 2019 Kernel Summit, Shuah Kahn [asked](<https://lwn.net/ml/ksummit-discuss/7b73e1b7-cc34-982d-2a9c-acf62b88da16@linuxfoundation.org/>) whether the community should adopt some sort of change-ID convention to track work heading into the kernel. Doug Anderson [proposed something similar](<https://lwn.net/ml/ksummit-discuss/CAD=FV=UPjPpUyFTPjF-Ogzj_6LJLE4PTxMhCoCEDmH1LXSSmpQ@mail.gmail.com/>) a month later. The [extended discussions](<https://lwn.net/Articles/797613/>) that followed did not lead to the adoption of a change ID, but they did bring about a related change. 

Specifically, Thomas Gleixner [suggested](<https://lwn.net/ml/ksummit-discuss/alpine.DEB.2.21.1906290802360.1802@nanos.tec.linutronix.de/>) the use of a Link tag that would contain an archive URL for the posting of each commit on the mailing lists: 

> What's really useful is when the commit has a Link tag: 

```
>        https://lore.kernel.org/lkml/$MESSAGE-ID
>
```

> 
> and if the submitters provide the same kind of link in their V(N) submission pointing to the V(N-1) in the cover letter: 

```
>         https://lore.kernel.org/lkml/$MESSAGE-ID-V(N-1)
>
```

> 
> If it's a single patch the link can be in the patch itself after the --- separator. That allows a quick lookup of the history. 

The Link tag was not new; its first appearance in the kernel repository is in [this 2011 commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f994d99cf140>). That commit, along with 56 others containing Link tags, went into the 2.6.39 release. This tag was mostly used for commits to the x86 and core-kernel subsystems initially, showing up in 3-500 commits per release. As this plot shows, the use of Link tags grew slowly over time, until something happened: 

> ![\[Plot of link tags per release\]](https://static.lwn.net/images/2025/linktags.png)

The "something" that happened partway through the 5.2 development cycle was, of course, the above-mentioned discussion. There seemed to be widespread agreement that the addition of Link tags to commits would be helpful. Kees Cook posted [a Git hook configuration](<https://lwn.net/ml/ksummit-discuss/201907021638.3D0AC00@keescook/>) that would add the tag automatically whenever a patch was applied with `git am`. Linus Walleij [updated the kernel documentation](<https://git.kernel.org/linus/291084904eb0>) to suggest using this hook; the [b4 tool](<https://b4.docs.kernel.org/en/latest/>) later gained a flag to add Link tags as well. The addition of Link tags grew accordingly; in the 6.16 release, 11,030 commits (75% of the total) included Link tags. 

(It is worth noting that the other part of Gleixner's suggestion — including a link to the previous posting whenever a patch series is updated — has not been adopted as widely, though many developers do indeed include those links. These backward links are important, as any LWN kernel writer will attest, if one wishes to look at the history of a change through multiple iterations.) 

During the 2019 discussion, Torvalds [was lukewarm to the idea](<https://lwn.net/ml/all/CAHk-=wgqemMJqX4SzbK52KicWSiK4_1qUus=q1akkwdEqXOkvQ@mail.gmail.com/>) of including Link tags; he certainly did not oppose it, and said it was better than trying to create some sort of change ID. By 2022, though, he was [beginning to complain](<https://lwn.net/ml/all/CAHk-=wjMmSZzMJ3Xnskdg4+GGz=5p5p+GSYyFBTh0f-DgvdBWg@mail.gmail.com/>) about them: 

> I _really_ wish the -tip tree had more links to the actual problem reports and explanations, rather than links to the patch submission. 
> 
> One has the actual issue. The other just has the information that is already in the commit, and the "Link:" adds almost no actual value. 

This refrain would become more common over the years, culminating in some strongly worded complaints in the middle of discussions on [virtual filesystem layer](<https://lwn.net/ml/all/CAHk-=whBm4Y=962=HuYNpbmYBEq-7X8O_aOAPQpqFKv5h5UbSA@mail.gmail.com>) and [io_uring](<https://lwn.net/ml/all/CAHk-=wjamixjqNwrr4+UEAwitMOd6Y8-_9p4oUZdcjrv7fsayQ@mail.gmail.com>) patch sets. The core problem remained the same: he does not like it if he follows the URL in a Link tag, hoping to learn more about the change in question, and only finds the same information that is already contained in the changelog itself. That slows his workflow down and increases his grumpiness level. 

Various developers have sought to defend the use of these tags in these discussions. Christian Brauner [said](<https://lwn.net/ml/all/20250826-umbenannt-bersten-c42dd9c4dc6a@brauner>): ""I care that I can git log at mainline and figure out where that patch was discussed, pull down the discussion via b4 or other tooling, without having to search lore"". Konstantin Ryabitsev [pointed out](<https://lwn.net/ml/all/20250827-military-grinning-orca-edb838@lemur>) that it is not always easy to find patch submissions by searching the archives. Jens Axboe [said](<https://lwn.net/ml/all/f0f31943-cfed-463d-8e03-9855ba027830@kernel.dk>) that the tags can help to find the cover letter for a patch series, and that they can also help to turn up discussion that happens after a patch is applied. Greg Kroah-Hartman [argued](<https://lwn.net/ml/all/2025090901-mangle-provable-6248@gregkh>) for keeping the tags, saying ""they work well for those that have to spelunk into our git branches all the time"". 

Torvalds, though, has been unmoved by these arguments and steadfastly opposed to the use of Link tags except in cases where there is something "interesting" behind the link. In the recent discussions, Axboe [asked](<https://lwn.net/ml/all/51f10765-1d74-48dd-8d5b-76178cf5dc66@kernel.dk>) repeatedly for a proclamation from Torvalds on what the rules for Link tags should actually be (and [suggested summoning LWN](<https://lwn.net/ml/all/a65abd25-69bd-4f10-a8b8-90c348d89242@kernel.dk>) for a summary). Torvalds appears to have answered that request in the notes for the [6.17-rc5](<https://lwn.net/Articles/1037071/>) release: 

> So if a link doesn't have any extra relevant information in it, just don't add it at all in some misguided hope that tomorrow it will be useful. 
> 
> Make "Link:" tags be something to celebrate, not something to curse because they are worthless and waste peoples time. 
> 
> Please? 

Many maintainers are unlikely to celebrate the fact that they have to end the automatic addition of Link tags and think, for each commit, whether such a tag is "interesting" or not. But they can celebrate the fact that the time spent on that exercise will save some time responding to grumpy emails from the Chief Penguin. While the new policy may not be entirely popular among maintainers, there is at least now something approaching an actual policy around the use of these tags.
