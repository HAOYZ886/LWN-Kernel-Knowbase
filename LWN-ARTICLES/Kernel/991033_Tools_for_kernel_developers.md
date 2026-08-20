---
title: Tools for kernel developers
url: https://lwn.net/Articles/991033/
date: "September 23, 2024"
category: Development tools
author: "By Jonathan Corbet September 23, 2024 Maintainers Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 23, 2024

* * *

[Maintainers Summit](<https://lwn.net/Articles/990740/>)

Konstantin Ryabitsev started a session on development tooling at the 2024 Maintainers Summit by saying that he does not want to be a "wrecking ball". If a given workflow is working for people, he does not want to try to force any sort of change. That said, he has ideas for how he can continue his work on providing better tooling for the development community. 

[![\[Konstantin
Ryabitsev\]](https://static.lwn.net/images/conf/2024/ms/KonstantinRyabitsev-sm.png)](<https://lwn.net/Articles/991056/>) The use of the [b4 tool](<https://b4.docs.kernel.org/en/latest/>) is increasing, he said, and the new features for patch preparation and submission have been well received. He is working on a `b4 review` command that will ease the manual process of sending Acked-by and Reviewed-by tags for patches written by others. Jason Gunthorpe asked whether it will be possible to send free-form remarks along with the tags; the answer was "yes". 

Another area of work is the "bugspray" bot that is intended to integrate the [kernel bugzilla instance](<https://bugzilla.kernel.org/>) with the project's mailing lists and Git repositories. Use of bugspray within a given subsystem will require maintainer buy-in; those who do not like bugzilla are free to ignore it. Since bugspray can look at a report and try to figure out which subsystems are involved, Ryabitsev is thinking about removing all of the various subsystem components from the bugzilla server. Many of those components have not been used in years, but it is hard to tell which ones are active, and users can have difficulties choosing the correct one. Instead, users would just file bugs for the kernel and the bot would figure it out. 

One thing that is needed, he said, is volunteers to be the first point of contact for bug reports. Thorsten Leemhuis said that he would be willing to do that if he could find funding for the work. 

Ted Ts'o said that he needs the ability to move bugs between subsystems; often, as developers dig into why something is going wrong, they conclude that the bug is not where it first seemed to be. So, to be useful, bugspray needs to make it possible to reassign a bug. 

Leemhuis pointed out a problem with the existing bugzilla instance. Email addresses, which must be provided to file a bug, are considered to be private information. Using a bug submitter's address to include them in email conversations is thus a GDPR violation, and evidently somebody has complained. Linus Torvalds said that there needs to be a public notice on the bugzilla site that email addresses are public; a similar thing was done with the kernel's [developer certificate of origin](<https://docs.kernel.org/process/submitting-patches.html#sign-your-work-the-developer-s-certificate-of-origin>) years ago. 

With regard to the [lore email archive](<https://lore.kernel.org>), Ryabitsev is working on providing pre-filtered email inboxes for specific subsystems. The Linux Foundation continues to fund work on the public-inbox archive system that underlies lore. There is experimental work on an automated "what's new" summary generator that could provide an overview of what is happening in a given subsystem. This feature uses a large language model to generate the summaries, and is "hit and miss" for now, he said. 

There are early trials underway to provide development-forge services using [Forgejo](<https://forgejo.org/>); nothing is publicly available yet. 

What Ryabitsev wanted more than anything else was to find out if the maintainers in the room support the work that he is doing. Along with everything described above, that work includes maintaining the lore email archives, writing workflow documentation, helping new maintainers get started with the tools and processes, deploying new services, and maintaining the kernel keyring. The answer to that question was clear: the development community truly appreciates this work, and would like to see a lot more of it. 

As things wound down, Leemhuis said that he would like to get his [regzbot](<https://gitlab.com/knurd42/regzbot>) regression-tracking tool on the list of supported systems. It can do things that bugzilla cannot, he said, including monitoring outside trackers for information on regressions. There was some unfocused talk on the maintenance of the [patchwork](<https://en.wikipedia.org/wiki/Patchwork_\(software\)>) system and whether it still needs to exist. Ts'o asked for an authentication mechanism that would allow automated systems (such as continuous-integration testers) to get past kernel.org's increasingly fortified bot defenses. 

At the close of the session, Ryabitsev said that he was happy to continue working to support the community, but that he could use more help. Keeping the email archives going, in particular, is a surprisingly labor-intensive task that makes it hard to get anything else done.
