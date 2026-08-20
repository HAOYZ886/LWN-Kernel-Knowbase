---
title: "\"git request-pull\" and confusing diffstats"
url: https://lwn.net/Articles/802797/
date: "October 21, 2019"
category: "Development tools-Git"
author: "By Jonathan Corbet October 21, 2019"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 21, 2019

When a kernel subsystem maintainer has a set of commits to send up the chain toward the mainline, the [`git request-pull` command](<https://git-scm.com/docs/git-request-pull>) is usually the right tool for the job. But various maintainers have noticed over the years that this command can sometimes generate confusing results when confronted with anything but the simplest of histories. A brief conversation on the linux-kernel mailing list delved into why this situation comes about and what maintainers can do in response. 

While `git request-pull` is intended to be a general-purpose command, it's no secret that its output is designed for the needs of one specific consumer; it contains almost everything Linus Torvalds needs to know when considering a request to pull a tree into the mainline kernel. This information includes the commit upon which the tree is based, the timestamp for the most recent commit in the tree, the tree to pull the commits from, the maintainer's description of the changes, a list of commits (in the `git shortlog` format), and a diffstat showing which files will be touched. See [a typical pull request](<https://lwn.net/ml/linux-kernel/20190709090611.66911ed5%40lwn.net/>) to see how all of those elements are worked into a single email. 

That example is generated from a relatively straightforward development history that looks something like this: 

> ![\[commit stream\]](https://static.lwn.net/images/2019/commits1.svg)

Generating both the commit log and the diffstat for this history is relatively straightforward, and the pull requests looks exactly as one would expect. 

Recently, Will Deacon [ran into a more complex situation](<https://lwn.net/ml/linux-kernel/20191017234348.wcbbo2njexn7ixpk@willie-the-truck/>), though. His tree was initially based in 5.4-rc1, but then required a merge of 5.4-rc2 to obtain the dependencies for a fix. The history ended up looking something like this: 

> ![\[commit stream 2\]](https://static.lwn.net/images/2019/commits2.svg)

When one runs `git request-pull` for a tree like this, the commit-log portion will look exactly as expected — it contains the commits in the local tree. The diffstat, though, is likely to reflect a large number of unrelated changes, making the pull request look like a scary beast indeed. In essence, that diffstat will reflect not just the local changes, but also everything that was pulled into the local tree when 5.4-rc2 was merged. That makes it hard, at best, to see what the pull request will actually do. 

Deacon was [surprised](<https://lwn.net/ml/linux-kernel/20191018174153.slpmkvsz45hb6cts@willie-the-truck/>) that the commit log was correct while the diffstat was wrong. Torvalds [explained](<https://lwn.net/ml/linux-kernel/CAHk-=whmtB98b8=YL2b8HzPKRadk2A9pL0aasmvgebhePrDP9w@mail.gmail.com/>) it this way: 

So logs and diffs are fundamentally different. 

A log is an operation on a _set_ of commits (that's the whole point \- you don't list the beginning and the end, you list all the commits in between), while a diff is fundamentally an operation on two end-points and shows the code difference between those two points. 

And the summary versions of those operations (shortlog and diffstat) are no different. 

He went on to say that, when there are only two endpoints and the history is simple, it is not hard for a tool like Git to calculate the difference between them. Throwing another merge into the mix complicates the situation, though, by adding another endpoint. The end result is the useless diffstat included in the pull request. 

Deacon resolved the issue by merging the current mainline with the tree to be pulled: 

> ![\[commit stream 3\]](https://static.lwn.net/images/2019/commits3.svg)

The diffstat could then be generated manually choosing the mainline and the merged tree as the two endpoints; the only differences that will be visible will be those that are not in the current mainline — the changes to be pulled from Deacon's tree, in other words. The clean diffstat was then manually patched into the pull request. The merge itself can then be thrown away; it should not be part of the commit stream sent upstream. As Torvalds explained, performing the merge reduced the diffstat problem back to two simple endpoints, so the result became as one would expect. 

Should maintainers perform this sort of dance before sending pull requests upstream? It seems that Torvalds appreciates the effort; he [described](<https://lwn.net/ml/linux-kernel/CAHk-=wjPZYxiTs3F0Vbrd3kRizJGq-rQ_jqH1+8XR9Ai_kBoXg@mail.gmail.com/>) the result as a ""good quality"" pull request. He also noted, though, that he often gets pull requests with confusing diffstats and doesn't really have a hard time dealing with them. Still, maintainers who want to be sure that their pull requests are as pleasing to the recipient as possible may want to go to the extra effort. 

The best solution, of course, would be to fix `git request-pull` to do the right thing in this sort of situation. Depending on how complex the merge history is, though, "the right thing" may not always be entirely obvious. It might also, like the merge described above, require changing the repository, which the `request-pull` command does not currently do. But, as Ingo Molnar [noted](<https://lwn.net/ml/linux-kernel/20191021064658.GB22042@gmail.com/>), it should at least be possible for `git request-pull` to detect this situation and issue a warning when it arises. Then, at least, developers would not be surprised by a bogus diffstat — something that can easily happen immediately after having sent it upstream.
