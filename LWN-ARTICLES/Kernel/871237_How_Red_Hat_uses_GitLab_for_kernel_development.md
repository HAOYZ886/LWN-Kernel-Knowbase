---
title: How Red Hat uses GitLab for kernel development
url: https://lwn.net/Articles/871237/
date: "October 1, 2021"
category: "Development tools-Forges"
author: "By Jonathan Corbet October 1, 2021 LPC"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 1, 2021

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2021-Linux_Plumbers_Conference>)

Much of the free-software development world has adopted Git forges (such as GitHub, GitLab, or sourcehut) with enthusiasm. The kernel community has not. Reasons for that reticence vary, but one that is often heard is that these forges simply don't work well at the scale needed for the kernel project. At [a Kernel-Summit session](<https://linuxplumbersconf.org/event/11/contributions/988/>) during the [2021 Linux Plumbers conference](<https://linuxplumbersconf.org/>), Donald Zickus and Prarit Bhargava sought to show how Red Hat has put GitLab to good use to support its kernel team. Not only can these forges work for kernel development, they said, but moving to a forge can bring a number of advantages. 

#### The transition

Red Hat has transitioned its kernel team from "an old Patchwork server" to GitLab in the last year, Zickus began. Prior to the change, the team had a fairly traditional, email-based workflow that got harder to manage as the patch volume increased. Red Hat has a number of strict rules regarding patch review and getting acknowledgments from the appropriate people; tracking the readiness of patches as they went through this gauntlet, which involves a lot of manual work, became increasingly hard. Reviewers didn't know which patches they should be looking at, and the continuous-integration (CI) system was bolted on. 

It was time to make a change, so the company turned to GitLab. 

Bhargava briefly introduced the [Lab tool](<https://github.com/zaquestion/lab>), which provides a command-line interface to many GitLab features. Perhaps ironically, this tool is hosted on GitHub. A lot of developers prefer a command-line interface, he said. 

In general, kernel maintainers tend to have their own scripts; each maintainer's tooling is different. Some maintainers would detect certain kinds of errors, while others would not. GitLab's ability to run scripts on actions has replaced much of this customization, ensuring that each patch is treated consistently and has the proper signoffs, includes the (apparently mandatory) Bugzilla ID, etc. Patches that come up short in one way or another can be labeled as needing attention. 

Email, Bhargava said, makes it easy to comment on patches. It is rather less easy for maintainers to sift through the resulting volume of messages. GitLab is able to thread comments and replies, all organized per merge request, making the process easier. All of this is tied to a "big fat 'approve' button" to allow a merge to proceed. At this point, he said, he's not seeing developers using email-based approvals anymore. 

The upstream kernel uses the `MAINTAINERS` file to determine who should review a given patch, which is another step for contributors to remember. Within Red Hat, that process has now been automated; when a merge request is generated, maintainers and reviewers are assigned automatically based on an `owners.yaml` file. There are two categories of review, depending on whether the reviewer's approval is required. Interested developers can sign up for notifications for changes to specific areas. 

Previously, CI was added to the email-based process, separated from the generation of patches. Nothing required its use. In the new system, CI is integrated directly. While CI systems are not exciting to most maintainers, he said, they should be; they add a lot of stability to the kernel. With some of the CI testing done for the kernel as a whole, developers don't even know that the testing is happening; GitLab makes CI testing explicit and visible. 

Zickus took over to say that the experience with GitLab has not been entirely smooth; they have found various problems over time. GitLab has worked with them to resolve these problems, which were mostly with the API and tooling. Red Hat also has a dedicated group working on issues that GitLab has not been able to resolve; there is a "strategic partnership" between the two companies. 

There are some open issues, of course, including managing the chain of trust: pull requests for the kernel need to be properly signed. Better logs for merge requests would be helpful. Perhaps the biggest concern, though, has to do with making GitLab into a single point of failure; what if the company is bought out by somebody who is hostile to Red Hat and its goals? In that case, it would be relatively easy to pull all of the necessary data out of the system; the Git trees are already mirrored elsewhere. They have a script now that can take all of the comments from GitLab and dump them into a [public-inbox](<https://lwn.net/Articles/748184/>) instance. 

Prarit closed the prepared talk by saying that it is still not 100% clear that GitLab is the best way forward, even though Red Hat is fairly deeply invested in it at this point. But the Git forge approach is worthwhile, he said. There were a lot of worries about making this kind of transition that turned out not to be real problems. 

#### Discussion

Greg Kroah-Hartman started the discussion by (somewhat jokingly) congratulating the speakers for having integrated all of [Gerrit](<https://www.gerritcodereview.com/>)'s functionality into GitLab. But, he asked, how many patches are they really managing this way? Zickus answered that they manage 15-16,000 patches for each RHEL update, every three months. Bhargava said that, when they sat down to look at forges two years ago, Gerrit didn't have many of the features they were looking for; perhaps it has gotten better since. 

Ted Ts'o said that his biggest worry is collaboration between subsystems. If information for one subsystem is "orphaned" in GitLab, that is going to make life harder for developers elsewhere. He is worried about discussions in particular; comments in Gerrit, for example, are not available anywhere else. The kernel community could consider hosting a GitLab instance at, [![\[Discussion\]](https://static.lwn.net/images/conf/2021/lpc/rh-gitlab-sm.png)](<https://lwn.net/Articles/871252/>) say, kernel.org, but then some patch comments would live there, while others would be on the mailing lists. Developers would have to search in two places to get the full picture. Unless email can remain a first-class citizen in the development process, he said, a workable transition is hard to see. 

Zickus answered that Red Hat is running an email bridge now that is used to ease the transition for developers. It is not intended to be kept around as a long-term solution, though. 

Konstantin Ryabitsev said that he is not excited about the possibility of hosting a forge instance on kernel.org. Hosting the Bugzilla instance there has not been a good experience; it is mostly abandoned, but he is stuck cleaning out the spam that accumulates there. Once a tool is added, it is almost impossible to remove, since a couple of people inevitably become dependent on it. So it will have to be maintained forever. 

A bigger issue, though, has to do with robustness. If kernel.org is not reachable, kernel development still goes on; an outage is inconvenient but not really a big problem. Adding a central forge, though, risks creating a situation where, should it go down, no work can get done. Imagine a situation, he said, where there is a zero-day vulnerability affecting billions of devices, and an attacker wants to prevent it from being patched. Attacking a crucial piece of infrastructure like a central forge might then look like a good idea. If the answer is "fall back to email", then nothing has really been solved. 

Zickus said that GitLab is replicated and stored on the Google cloud, to which Ryabitsev responded that the Google cloud is unavailable in parts of the world like Russia and China. A large cloud provider is not a good solution, he said; on the other hand, self-hosting brings its own set of scaling and expense problems. Zickus said that Red Hat has a large testing team in China, and GitLab works well there. If that were to change, though, it would be a real problem. 

Ryabitsev said he is not opposed to subsystems switching to a forge, as long as it doesn't become "a place where discussions go to die". Currently, if the infrastructure goes away, the "fossil record" is still around on sites like lore.kernel.org. Zickus said that dumping conversations to a public-inbox solves some of that problem. He then summarized the conversation by saying that there isn't really opposition to using the tool, but there are worries about preserving the conversations that happen there. 

As time ran out, Ts'o said that this session was not the end of the conversation. Forges like GitLab point out what could be in our future, and that features like automatic CI testing are a really good idea. 

The [video of this talk](<https://www.youtube.com/watch?v=gpFfJkrrEEs&list=PLVsQ_xZBEyN2c21jFUgqI2iMa094zXanH&index=37&t=31m50s>) is available on YouTube.
