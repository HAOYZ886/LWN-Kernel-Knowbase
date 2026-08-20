---
title: "An LSFMM development-process discussion"
url: https://lwn.net/Articles/932215/
date: "May 23, 2023"
category: Development model
author: "By Jonathan Corbet May 23, 2023 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 23, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Hannes Reinecke led a plenary session ostensibly dedicated to the "limits of development". The actual discussion focused on the frustrations of the kernel development process as experienced by both developers and maintainers. It is probably fair to say that no problems were solved here, but perhaps the nature of some of the challenges is a bit more clear. 

Reinecke started with the topic of patch sets that are proposed by their author, but do not end up being either applied or rejected. Or perhaps a patch set will be rejected after the developer has put the work into posting numerous revisions. Meanwhile, other patches that are seemingly just as intrusive are accepted without problems. This unpredictability makes it hard for companies — even those that are relatively deeply involved with the kernel community — to know what will happen when they start on a development project. 

Nobody wants to spend a lot of time on a project that will end up being rejected. He hasn't been able to come up with guidelines for these developers other than to say that, if they approach the maintainer at the wrong time, their work will not get in. This seems somewhat inadequate, so he would like to find a way to apply the brakes early for work that has fundamental problems and will not find acceptance. 

Josef Bacik said that the Btrfs community has put some effort into addressing this problem. As the patch counts have gone up, the community has gotten better at figuring out how and when patches go in; this took a lot of effort from the maintainers and technical leaders. An effort has been made to provide early indications of whether a project is worth doing and, if it is, to put in the work to shepherd the project along. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

A lot of kernel maintainers, he continued, see their subsystem as their own fiefdom and don't care much about work from other developers. Meanwhile, drive-by authors don't know who they should be paying attention to; this is a fundamental problem in our community. Maintainer behavior can be erratic and inconsistent. We should be able to work better together, he said. Getting there requires a more clear definition of what the maintainer's role is. It is not just accepting patches. A maintainer should provide clear indications to newcomers, provide feedback on ideas, and ensure that patches get reviews. The Btrfs developers get together regularly to talk about their work, but there is no equivalent forum for the wider community. 

Steve Rostedt mentioned the kernel's communication-style documentation, which is currently being (slowly) rethought by the Linux Foundation Technical Advisory Board. Our documentation on how maintainers talk to developers can stand improvement, but there is also a place for a document on how contributors should talk to maintainers. Reinecke answered that all of this implies that people actually talk to each other. 

Bacik said that communication style matters. Anybody who has been in his presence for long knows that he tends toward the use of profanity when he speaks. But he has made a point of not doing that on the mailing lists; he never knows how it will be perceived, especially by people who do not know him. It could cause contributors to decide to stay away from the community. We should all make a similar effort to be more welcoming, he said. He also expressed unhappiness about people who continue to fight over things that have been settled; he mentioned BPF and systemd as examples. We are all moving in the same direction, he said, and should pretend that we like each other. 

Reinecke asked if the problem is just one of communication. A pattern he has often seen is where the relevant parties enter a discussion late, then dig in their heels. Bacik said that it's the maintainer's job to steer a project; that includes looking at random patch sets and giving feedback. A flat refusal to accept a patch is not helpful; the author had a use case in mind and didn't write the code just for fun. They are trying to accomplish something, and the maintainer should make the effort to understand why. 

Ted Ts'o said that running the project has to be a team effort; if the onus is put just on the maintainers, they will burn out. Maintainers simply cannot review all of the patches that hit the lists; they are dependent on others to help with that work. One cannot demand a service-level agreement from a volunteer maintainer. Few maintainers, he said, do that work as a full-time job. 

Kent Overstreet said that developers want quick feedback. Reviewers often think that every review has to be highly detailed; that is demanding and slows down the process. Christian Brauner answered that some patches really do require deep review, and that just doesn't scale. Sometimes, he said, we just have to accept code that is not perfect. 

Brauner said that he would like to see more patches with Reviewed-by tags. In his experience, potential reviewers worry that their tag will make them look bad if a bug is found in the patch later on. But bugs happen, he said, and they don't mean that a reviewer hasn't done their job. Dan Williams said that we are all managing our reputations in the community, and that can cause people to hesitate; maintainers need to engage with contributors, but the contributors also have to give the maintainers some space. Maintainers, he said, should talk to each other more and not be afraid to ask for help. There is also value in having regular phone calls. Most of a project's communication channels are public and archived; having a more private channel can put people at ease. 

James Bottomley asked whether the community is sufficiently encouraging of reviews. In the past there have been problems with people giving bad advice, leading the community to clamp down somewhat. But perhaps the correction has gone a little too far? Brauner said that a maintainer will only get reviews if they encourage reviewers; if you see somebody who is doing good work, write to them and ask them to continue. We are good at saying something is wrong, he said, but not so good at saying when something is right. 

As the session ran out of time, Bottomley asked how a potential reviewer might get the "R" tag in the `MAINTAINERS` file that will cause them to be copied on patches. In most subsystems, it seems that the reviewer has to explicitly ask for it. 

At that point, the session came to a close. It is not clear that a whole lot was achieved in the discussion, but it did at least give maintainers a chance to talk about their frustrations for a bit.
