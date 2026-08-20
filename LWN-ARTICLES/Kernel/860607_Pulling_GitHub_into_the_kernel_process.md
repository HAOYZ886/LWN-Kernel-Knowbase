---
title: Pulling GitHub into the kernel process
url: https://lwn.net/Articles/860607/
date: "June 23, 2021"
category: "Development tools-Forges"
author: "By Jake Edge June 23, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 23, 2021

There is an ongoing effort to "modernize" the kernel-development process; so far, the focus has been on [providing better tools](<https://lwn.net/Articles/811528/>) that can streamline the usual email-based workflow. But that "email-based" part has proven to be problematic for some potential contributors, especially those who might want to simply submit a small bug fix and are not interested in getting set up with that workflow. The project-hosting "forge" sites, like GitHub and GitLab, provide a nearly frictionless path for these kinds of one-off contributions, but they do not mesh well—at all, really—with most of mainline kernel development. There is some ongoing work that may change all of that, however. 

Konstantin Ryabitsev at the Linux Foundation has been spearheading much of this work going back at least as far as his [September 2019 draft proposal](<https://lwn.net/ml/ksummit-discuss/20190912120602.GC29277@pure.paranoia.local/>) for better kernel tooling. Those ideas were discussed at the [2019 Kernel Maintainers Summit](<https://lwn.net/Articles/799134/>) and at a [meeting at Open Source Summit Europe 2019](<https://lwn.net/Articles/803619/>) in October. Throughout, Ryabitsev has been looking at ways to make it easier for non-email patch submitters; along the way, he has also released the [`b4` tool for collecting up patches](<https://people.kernel.org/monsieuricon/introducing-b4-and-patch-attestation>) and worked on [patch attestation](<https://lwn.net/Articles/813646/>). 

A recent [post](<https://lwn.net/ml/workflows/20210616171813.bwvu6mtl4ltotf7p@nitro.local/>) to the kernel workflows mailing list shows some progress toward a bot that can turn a GitHub pull request (PR) into a well-formed patch series to send to the proper reviewers and mailing lists. ""This would be a one-way operation, effectively turning Github into a fancy 'git-send-email' replacement."" He also laid out some of the benefits that this bot could provide both for maintainers and patch submitters: 

>   * submitters would no longer need to navigate their way around git-format-patch, get_maintainer.pl, and git-send-email -- nor would need to have a patch-friendly outgoing mail gateway to properly contribute patches 
>   * subsystem maintainers can configure whatever CI pre-checks they want before the series is sent to them for review (and we can work on a library of Github actions, so nobody needs to reimplement checkpatch.pl multiple times) 
>   * the bot should (eventually) be clever enough to automatically track v1..vX on pull request updates, assuming the API makes it straightforward 

He had some questions about whether the bot should be centralized in a single repository (per forge platform) that would serve as the single submission point, or whether subsystem maintainers would want to configure their own repositories. The latter would give maintainers the opportunity to set their own criteria for checks that would need to pass (e.g. `checkpatch.pl`) before the PR was considered valid, but would mean that they might have to ride herd on the repository as well. 

In addition, Ryabitsev wondered when and how PRs would get closed. The bot could potentially monitor the mainline and auto-close PRs once the patch set gets merged, but that won't be perfect, of course. An easier approach for him would be ""to auto-close the pull request right after it's sent to the list with a message like 'thank you, please monitor your email for the rest of the process'"", but he was unsure if that would be best. 

As might be guessed, reactions from those participating in the thread were all over the map. While there is a lack of many kinds of diversity within the kernel community, opinions on development workflow—opinions, in general, in truth—do not have that problem. Some maintainers have zero interest in this kind of effort at all. As Christoph Hellwig [put it](<https://lwn.net/ml/workflows/YMsGoI0UdbmUebhX@infradead.org/>): ""Please opt all subsystems I maintain out of this crap. The last thing I need is patches from people that can't deal with a sane workflow."" 

Hellwig's complaint, which Jiri Kosina [agreed with](<https://lwn.net/ml/workflows/nycvar.YFH.7.76.2106171031220.18969@cbobk.fhfr.pm/>), may be more about the expectations of those who use GitHub (and the like), and less about the possibility of having a web-based interface to kernel development. Dmitry Vyukov [asked](<https://lwn.net/ml/workflows/CACT4Y+YoHYCKaLdZFSLqKsMz1j72i=k12FVpmSo-OyB7rCorsw@mail.gmail.com/>) why Hellwig and Kosina would be unwilling to accept patches from the system if they cannot really distinguish them from a regular submission. Vyukov said that he is currently experiencing a Git email submission problem that he is uninterested in working around, so he can see why others might be similarly inclined. Meanwhile, though, he sees benefits from this kind of bot: 

> On the other hand this workflow has the potential to ensure that you never need to remind to run checkpatch.pl, nor spend time on writing code formatting comments and re-reviewing v2 because code formatting will be enforced, etc. So I see how this is actually beneficial for maintainers. 

Hellwig is [not opposed](<https://lwn.net/ml/workflows/YMsfXuykH6z0ZbpM@infradead.org/>) to a web-based solution, though he wants nothing to do with GitHub. But Ryabitsev [seems uninterested](<https://lwn.net/ml/workflows/20210617145728.nahkvtxapozccm6c@nitro.local/>) in ""reimplementing a lot of stuff that we already get 'for free' from Github and other forges"". Both [Mark Brown](<https://lwn.net/ml/workflows/20210617151659.GF5067@sirena.org.uk/>) and [Laurent Pinchart](<https://lwn.net/ml/workflows/YMtpHPaJzMxmhslt@pendragon.ideasonboard.com/>) suggested that there are mismatches between GitHub-normal practices and those of the kernel community. Pinchart mentioned the inability to comment on a patch's commit message on GitHub as something that generally leads to poor messages; the platform is training these developers to a certain extent: 

> Developers who have only been exposed to those platforms are very likely to never have learnt the importance of commit messages, and of proper split of changes across commits. Those are issues that are inherent to those platforms and that we will likely need to handle in an automated way (at least to some extent) or maintainers will become crazy [...] 

But Miguel Ojeda [thinks](<https://lwn.net/ml/workflows/CANiq72n-L=aroXFfnoF96Edk6F7+hqHu_BTaKuKAeA0MgOcSbw@mail.gmail.com/>) that it is really no different from new developers showing up on the mailing list with patches. ""The same happens in the LKML -- some people have sent bad messages, but we correct them and they learn."" He also [noted](<https://lwn.net/ml/workflows/CANiq72kW+xsCWyhaT20a2=PUjUB59ujomaarnBrTDuYo1KEuJA@mail.gmail.com/>) that automated checking of patches can help both developers and maintainers: 

> [...] it is particularly useful to teach newcomers and to save time for maintainers having to explain things. Even if a maintainer has a set of email templates for the usual things, it takes time vs. not even having to read the email. 

Ojeda is [working](<https://lwn.net/Articles/860160/>) on the [Rust for Linux project](<https://github.com/Rust-for-Linux>), which we [looked at](<https://lwn.net/Articles/853423/>) back in April; he [said](<https://lwn.net/ml/workflows/CANiq72=qmGRTUYzXs_FP0Y2_pJOm=5O7bSTBwXawH29oTApuuw@mail.gmail.com/>) that he has also been working on a bot: 

> For Rust for Linux, I have a GitHub bot that reviews PRs and spots the usual mistakes in commit messages (tags, formatting, lkml vs. lore links, that sort of thing). It has been very effective so far to teach newcomers how to follow the kernel development process. 
> 
> I am also extending it to take Acks, Reviewed-by's, Tested-by's, etc., and then performing the merge only if the CI passes (which includes running tests under QEMU, code formatting, lints, etc.) after applying each patch. 

But Ojeda is taking things in a rather different direction than what Ryabitsev is envisioning. Ojeda wants to move the main place for patch review and the like from the mailing lists to GitHub. He is also considering having his bot pick up patches from the mailing list and turning them into GitHub PRs—the reverse of what Ryabitsev is doing. 

For his part, Ryabitsev [said](<https://lwn.net/ml/workflows/20210616182252.flsdrkjuw2etwcud@nitro.local/>): ""That's pretty cool, but I'm opposed to this on theological grounds. :)"" In particular, he is concerned about the "single point of failure" problem for the kernel-development infrastructure. If his bot is unavailable for any reason, it may be inconvenient for those who use it, but that will not hobble development. He sees GitHub as simply a ""developer frontend tool"". 

Somewhat similar to Ojeda's intentions, Brendan Higgins [has a tool](<https://lwn.net/ml/workflows/CAFd5g44cerCLF6yd_voFAEr9B+G4bLPve-ARVv8N30pHj9Zdzg@mail.gmail.com/>) to pick up patches from a mailing list (kselftest in this case) and upload them to a [Gerrit instance](<https://linux-review.googlesource.com/>). He sees some potential synergies between his bot and the one Ryabitsev is working on. Similarly, Drew DeVault [has been working on](<https://lwn.net/ml/workflows/CC57UCW5H2NE.3A6404HP4WC2Z@taiga/>) the reverse direction, from a mailing list to a project forge, as well. [Patchwork](<http://jk.ozlabs.org/projects/patchwork/>) is a longstanding code-review project that also collects up patches from mailing lists to populate a web application. It would seem that much of the focus is on getting patches _out_ of mailing lists, though, which is not where Ryabitsev is headed. 

While some maintainers want no part of this "GitHub Future", others are enthusiastic about the possibilities it could bring. Vyukov [thinks](<https://lwn.net/ml/workflows/CACT4Y+ZYG5EB3re0Lby920Atkmb7e_FthN_5XO881XOhF_iJzg@mail.gmail.com/>) that having a single GitHub repository with multiple branches will help consolidate the kernel-development landscape, which is currently fragmented on subsystem lines. He sees it as an opportunity to apply consistent coding-style standards; it does not matter which, he said, ""as long as it's consistent across the project"". It would also allow testing consistency throughout the tree and the same for the development process: 

> For once: it will be possible to have proper documentation on the process (as compared to current per-subsystem rules, which are usually not documented again because of low RoI [return on investment] for anything related to a single subsystem only). 

It is not at all clear that Vyukov's interest in consistency throughout the tree is shared widely, but there have certainly been complaints along the way about the difficulty of navigating between the different subsystem processes and requirements for submissions. There is also interest in making things easier for quick, one-off contributions; as Ryabitsev [put it](<https://lwn.net/ml/workflows/20210616180543.wevddhlzq4ow22f5@nitro.local/>): 

> Our code review process must also allow for what is effectively a "report a typo" link. Currently, this is extremely onerous for anyone, as a 15-minute affair suddenly becomes a herculean effort. The goal of this work is to make drive-by patches easier without also burying maintainers under a pile of junk submissions. 

Clearly keeping "junk submissions" to a bare minimum is going to be important. Linus Torvalds [said](<https://lwn.net/ml/workflows/CAHk-=whEZ23h_Gvg3aOuQgXbUjxn6gwvcS2dHawkPgpDwKmssw@mail.gmail.com/>) that he has had to turn off email from GitHub because it is too noisy; people have apparently signed him up as a project member without any kind of opt-in check. Beyond that, any kind of patch submission from PRs would need to have some sanity checks, including size limits, so that PRs like [one pointed out by Ryabitsev](<https://github.com/torvalds/linux/pull/779>) do not end up on the mailing list. 

That kind of PR highlights another problem: repository maintenance. Greg Kroah-Hartman [said](<https://lwn.net/ml/workflows/YMr5+ibXI5huLy7h@kroah.com/>) that there will be a need to monitor whatever repositories are being used for this purpose. It is not a small task: 

> What ever repo you put this on, it's going to take constant maintenance to keep it up to date and prune out the PRs that are going to accumulate there, as well as deal with the obvious spam and abuse issues that popular trees always accumulate. 

Torvalds does not want his GitHub tree used for this purpose and Kroah-Hartman said the same. However it plays out, someone will have to be tasked with keeping the repository tidy, which is ""a thankless task that will take constant work"". But Ryabitsev is [hopeful](<https://lwn.net/ml/workflows/20210617145934.jsbapuf65b5t54q4@nitro.local/>) that the Linux Foundation could fund that kind of work if it becomes necessary. 

In the end, it will likely come down to how seamlessly the GitHub bot fits in. If maintainers truly cannot really tell the difference in any substantive way, it is hard to see many of them rejecting well-formed patches that fix real problems in their subsystems. That ideal may not be reached right away, however, which might lead to a premature end to the experiment. It will be interesting to see it all play out over the coming months and years.
