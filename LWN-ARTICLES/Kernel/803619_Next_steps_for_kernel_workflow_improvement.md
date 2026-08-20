---
title: Next steps for kernel workflow improvement
url: https://lwn.net/Articles/803619/
date: "November 1, 2019"
category: Development tools
author: "By Jonathan Corbet November 1, 2019 OSS EU"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 1, 2019

* * *

[OSS EU](<https://lwn.net/Archives/ConferenceByYear/#2019-Open_Source_Summit_Europe>)

The kernel project's email-based development process is well established and has some [strong defenders](<https://lwn.net/Articles/702177/>), but it is also showing its age. At the [2019 Kernel Maintainers Summit](<https://lwn.net/Articles/799060/>), it became clear that the kernel's processes are much in need of updating, and that the maintainers are beginning to understand that. It is one thing, though, to establish goals for an improved process; it is another to actually implement that process and convince developers to use it. At the [2019 Open Source Summit Europe](<https://events19.linuxfoundation.org/events/open-source-summit-europe-2019/>), a group of 20 or so maintainers and developers met in the corner of a noisy exhibition hall to try to work out what some of the first steps in that direction might be. 

The meeting was organized and led by Konstantin Ryabitsev, who is in charge of kernel.org (among other responsibilities) at the Linux Foundation (LF). Developing the kernel by emailing patches is suboptimal, he said, especially when it comes to dovetailing with continuous-integration (CI) processes, but it still works well for many kernel developers. Any new processes will have to coexist with the old, or they will not be adopted. There are, it seems, some resources at the LF that can be directed toward improving the kernel's development processes, especially if it is clear that this work is something that the community wants. 

#### Attestation

Ryabitsev's first goal didn't feature strongly at the Maintainers Summit, but is an issue that he has been concerned about for some time: improving attestation for patches so that recipients can be sure of their provenance. Currently, there is no attestation at all, so recipients have to trust that patches come from the developer whose name appears on them. We all assume that maintainers are watching carefully and can catch spoofed emails, but the truth of the matter is that it is relatively easy to sneak malicious code past a maintainer. So an attacker could conceivably find a way to add a vulnerability to the kernel. 

The first problem to solve is thus, according to Ryabitsev, to fix attestation. Linus Torvalds does verify the signed tags that are associated with pull requests, he said, so that part of the process is taken care of. But there are no signatures on actual patches, and no consensus on how they might be added. 

His proposal is to introduce signatures on emailed patches as well. The mechanism used would be [minisign](<https://jedisct1.github.io/minisign/>), not [GnuPG](<https://www.gnupg.org>); one of the big advantages of minisign is that the attached signatures are much shorter than those created by GnuPG. Steve Rostedt interrupted at this point to question the value of this approach; he said that an attack, to be successful, would have to involve a relatively complex patch written in a style that mimics that of the purported author. It would be a big effort, he said; anybody with the resources to do that could also crack the encryption scheme used for attestation. 

Ryabitsev responded, though, that minisign is "real cryptography" and not easy to crack; there are far easier ways to get bad code into the kernel than breaking the encryption. The hard part with this scheme, instead, is with identity tracking. GnuPG, like PGP before it, is based on the "web of trust" idea, but the web of trust has proved to be mostly unworkable over the years and people are giving up on it. Newer schemes tend to be based, like SSH, on a "trust on first use" (or TOFU) model, where a new key is trusted (and remembered) when it is first encountered, but changes in keys require close scrutiny. He suggested using a TOFU approach in an attestation mechanism for Git as well. 

Rafael Wysocki was also skeptical, asserting that this scheme does not solve the problem; it only moves it elsewhere. An attacker could create an identity and build trust over time before submitting something malicious; the proposed scheme adds complexity but doesn't really fix anything, he said. Ryabitsev disagreed, though; building trust requires time and resources, but an attacker could spoof a trusted developer now. 

Frank Rowand asked whether maintainers would be expected to strip signatures before committing patches. The signature, Ryabitsev answered, would go below the "`‑‑‑`" line in the changelog, so it would be automatically stripped at commit time. But the key used would also be noted in a local database and verified the next time a patch shows up from the same developer. Rostedt pointed out that one-time submitters would not have a key in this database; Ryabitsev replied that, since those developers are not coming back, it doesn't really matter. This scheme is about trusting ongoing developers. 

He would like minisign-based attestation to become part of Git; tools like [`git format-patch`](<https://git-scm.com/docs/git-format-patch>) would just add it automatically. Rowand pointed out that a lot of developers use relatively old versions of Git, so it would take years to roll this capability out to everybody. He said that GnuPG should be used instead; developers have it and the kernel's web of trust already exists. But Ryabitsev said that GnuPG is a poor tool for signing patches; the attached signature is often larger than the patch itself, and list archival mechanisms tend to strip it out. To be truly useful, signatures on patches need to be unobtrusive. 

Like much of what was discussed in this meeting, signature use would be opt-in, at least initially. Ryabitsev is thinking about writing a bot that would watch the mailing lists and gently suggest to developers who are not using signatures that they might want to start. He asked the group whether this scheme as a whole was a good idea and got almost universal agreement (Rowand being the exception). So he intends to try to get the needed support added to Git. 

#### Base-tree information

A common question asked of patch submitters is: "which tree was this made against?". That information is often needed to successfully apply a patch, and CI systems need it to be able to do automatic testing. But that "base-tree information" is not included with patches currently; fixing that is high on many developers' wish lists. Dmitry Vyukov asked whether it would be better to add this feature to Git and wait for it to be adopted, or to create a wrapper script that developers could use now. It turns out, though, that the `‑‑base` option works in Git now, it's just a matter of getting submitters to use it. Vyukov agreed that this is the hardest part; he suggested creating a wrapper that would supply this option automatically. 

There was a bit of a side discussion on whether Torvalds would complain about the base-tree information, as he does when tags like [Change-id](<https://lwn.net/Articles/797613/>) show up in patches. The problem, though, is not really the extra tag, it's the perceived uselessness of the information. If the base-tree information is useful, there should not be complaints. 

It was pointed out that the base-tree information might not always be helpful to others; that base could be in a private tree, for example. At other times, though, it could be useful indeed. Rostedt pointed out that the "tip" tree used for x86 (and beyond) maintenance has a dozen or so branches in it; knowing which branch a patch applies to would be helpful. Everybody seemed to agree that this information should be there, and that the `checkpatch.pl` script should be made to check for it. There may eventually be a bot to nag developers who omit this information from their patches, but care would have to be taken to prevent it from generating too much noise. 

#### Beyond email

For a number of reasons, requiring all kernel patches to be sent by email looks like a policy with a limited future. Switching to a "forge" service, along the lines of GitHub or GitLab, is an idea without universal appeal, though, especially in the short term. But there is desire for a solution that could let some developers move beyond email while maintaining the current workflow overall. The first step in that direction is likely to be some sort of Git-to-email bridge. Ryabitsev pointed out, though, that there is no consensus on what such a bridge might look like. 

One option could be a special Git repository that developers could push to; any patch series pushed there would be turned into a series of emails and sent to the appropriate addresses. Ryabitsev does not like that idea, though; any such system would be a central point of failure that could go down at inopportune times. Another option would be some sort of web service that could be pointed at a public repository; once again, it would generate an email series and submit it. This solution falls down in another way, though: it is unable to support attestation. A third alternative is to create a command-line tool that can turn a pull request into an emailed series. 

There are a number of hard problems to be solved here, he said, with many tradeoffs to be considered. But the easiest solution appears to be the command-line tool, perhaps integrated with an tool like [GitGitGadget](<https://github.com/gitgitgadget/gitgitgadget>). There is also a tool under development at [sourcehut](<https://sourcehut.org/>) that is worth a look. He might support such a tool by exposing an SMTP service specifically for mailing patches to kernel.org addresses. 

That led to the concept of "feeds" — services that provide access to patches and more. The [lore.kernel.org](<https://lore.kernel.org>) service has been running for a while now; it has quickly become an indispensable part of the kernel development process. Ryabitsev would, though, like to create something with similar functionality that does not need a mailing list behind it. Developers could use it to create their own patch feeds; CI systems could also export feeds describing the tests they have run and the results. Then it would be possible to, for example, automatically annotate patches with data on how they have been tested and by who. Bots could use this information to figure out which tests they should run, avoiding those that have already been run elsewhere. Feeds would be archived and mirrored so they could be checked years in the future. Feeds would be able to support attestation, record `Acked-by` tags, and more. 

But that still leaves the problem of actually creating all of this tooling and making it easy to use. Nobody is going to want all of these feeds in their inbox, so it will have to be possible to establish filters. Size also matters: lore.kernel.org currently requires about 200GB of disk space, which is a bit unwieldy to download to one's laptop. But lore contains a lot of ancient history that developers will not normally need, so the database could be much smaller. 

Ryabitsev is currently working with the maintainer of [public-inbox](<https://lwn.net/Articles/748184/>) on the development of some of these tools. There is, he said, some development time that is available at the LF over the next six months; what should he aim to achieve in that time? Building something with Docker would be convenient for many, but the "old-school developers" don't want to deal with Docker. Should it be a command-line or web-based tool? Fans of command-line tools tend to be more vocal, but that does not mean that they are a majority. 

Perhaps, he said, the way to start would be to make it easy to set up a local [Patchwork](<http://jk.ozlabs.org/projects/patchwork/>) instance. There was a wandering discussion on how subsystems with group maintainership could be supported, but that is not a problem that can be solved in the next six months, he said. Further discussion on how the tools should be developed was deferred to [the kernel workflows mailing list](<https://lore.kernel.org/workflows/>). 

As time ran out there was some quick discussion of CI systems, including GitLab, Gerrit, and more. The kernel clearly needs more CI testing, so Ryabitsev wants to be sure that it is all integrated into any new tooling. He would like to be able to provide a feed describing what each of these systems is doing. These forge systems mostly provide an API for event data now; what is needed is a set of translator bots that could pull those events together into a public-inbox feed for anybody who is interested. CI systems would be able to consume this data, and others could follow it without having to have an account on each CI system. 

The emails sent by CI systems now are just noise to many recipients, he said; as more of these systems come online that problem will get worse. Creating a feed solves the problem by putting CI reports where only the people who want them have to see them. It is a hard thing to do well, he said, and he is not sure how his solution will work, but he wants to try. Email is a terrible way to integrate with systems that need structured data, so he's looking to replace the email message bus with a more structured, feed-based system. 

The session broke up with a statement that, if the community asks for this kind of tooling, there is a real possibility that the LF will find a way to fund its development. 

See also: [Han-Wen Nienhuys's notes](<https://docs.google.com/document/d/1khLOBw5-HyaaNX7xregpHQLSfvGDUeHDY921bkI-_os/edit>) from the meeting. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting your editor's travel to the event.]
