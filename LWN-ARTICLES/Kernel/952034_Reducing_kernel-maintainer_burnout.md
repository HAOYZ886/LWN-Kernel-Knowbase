---
title: "Reducing kernel-maintainer burnout"
url: https://lwn.net/Articles/952034/
date: "November 24, 2023"
category: Development model
author: "By Jonathan Corbet November 24, 2023 Maintainers Summit"
---

By **Jonathan Corbet**  
November 24, 2023

* * *

[Maintainers Summit](<https://lwn.net/Articles/951847/>)

Overstressed maintainers are a constant topic of conversation throughout the open-source community. Kernel maintainers have been complaining more loudly than usual recently about overwork and stress. The problems that maintainers are facing are clear; what to do about them is rather less so. A session at the 2023 Maintainers Summit took up the topic yet again with the hope of finding some solutions; there may be answers, perhaps even within the kernel community, but a general solution still seems distant. 

Ted Ts'o started off the session by saying that kernel maintainers end up having to do all of the tasks that nobody else working on a given subsystem wants to take on. These can include patch review, release engineering, testing, and responding to security reports. The expectations placed on maintainers have gone up over time, and kernel maintainers are feeling the pressure as a result. 

[![\[Ted Ts'o\]](https://static.lwn.net/images/conf/2023/lpc/TedTso-sm.png)](<https://lwn.net/Articles/952037/>) Darrick Wong, Ts'o continued, [broke down the aspects of the maintainer job](<https://lwn.net/ml/linux-xfs/169116630379.3243794.7476300844644954184.stgit@frogsfrogsfrogs/>) nicely when he [stepped down](<https://lwn.net/ml/linux-xfs/169116629797.3243794.7024231508559123519.stgit@frogsfrogsfrogs/>) as the XFS maintainer. Ts'o is uncertain, though, about how well that has worked to get others to step up and tackle some of those jobs. 

Martin Petersen asserted that the real problem is that people are sending too many patches, but Dave Airlie strongly disagreed. As the DRM subsystem maintainer, he is "processing more changes than anybody" without having to touch a single patch. The way to handle this problem is to build up a structure with people who are able to take on the various tasks needed. The filesystem layer, he said, is more important than graphics; why doesn't it have more people than the DRM layer does? 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Josef Bacik answered that building up that structure is hard; he has been trying with various people for the last three years. One developer simply couldn't do the work, another was unable to bring things to a conclusion. The filesystem problem space is complicated, he said, and finding people who can work in that area is hard. 

Steve Rostedt said that part of the problem may be documentation; when he runs into bugs, he can't find documents describing how things work. Christoph Hellwig suggested writing down problems as they are encountered. Christian Brauner said that there is extensive documentation about the filesystem layers, but it tends to be hard to understand. 

Airlie said that the trick is for maintainers to leave voids for others to fill. Thomas Gleixner, though, brought up the example of the generic interrupt subsystem. There is currently one person maintaining it, even though "if it breaks, the whole world breaks". There are a lot of people sending patches, but nobody showing any desire to maintain it. Airlie said that, if there are 100 people sending patches, there may be five who can be convinced to help maintain the subsystem; Gleixner answered that he sees a lot of "random drive-by names" that clearly have no intention of sticking around. 

The need for reviewer help in particular came up; Linus Torvalds jumped in to say that reviewing is boring, so it is unsurprising that people don't want to do it. He keeps seeing huge patch sets being sent out once each week with little seeming to happen with them. A few tweaks, perhaps, before the next version is sent, but no real resolution. What is really needed, he said, is to find ways to get away from the email patch model, which is not really working anymore. He feels that way now, even though he is "an old-school email person". 

Bacik said that the Btrfs developers are using GitHub to track things; it is good at showing the work that is outstanding, so he can see what has been languishing. That has improved the subsystem's throughput significantly. There are, he said, tools out there that "make the tasks we hate easier", and that every other project uses to get their work done. Gleixner, though, expressed skepticism that adopting another tool would solve the problem. He has a patch-tracking system that works well enough, he said; the real solution is to teach managers that, with proper engineering, work gets done sooner (and life for maintainers is easier). 

Ts'o said that he never knows how long a patch submitter will be around, so it's never clear whether time spent to educate them will be worthwhile. He also said that, while asking submitters to fix existing technical debt in order to get their work merged is asking too much, maintainers can take a stand against adding more debt. Hellwig said that a developer trying to contribute code is often the best opportunity to get some cleanup done. Bacik said that the Btrfs community has been guiding developers that way for a long time and has learned how to do it well; he admitted that he should be writing their approach down. Maintainers should, he said, take a bigger role in teaching others. 

Gleixner said that a lot of useful information for contributors has indeed been written down. Dave Chinner, though, worried that pointing contributors to that documentation can come across as an impersonal brush-off. That is why he often takes the time to write a long response to contributors needing guidance. Rostedt said that today's developers have different needs. He asked how many in the room had started kernel work because their employers had told them to; no hands were raised. That is not the case for many of today's new developers, he said. 

"Being a maintainer is a part of our identity", Airlie said; it is likely how we got our current job and is not something that we readily let go of. Brauner added that people tend to hold onto power for dear life. Wolfram Sang said that he likes reviewing, but can get no support for doing that work; Dan Williams said that developers tend not to understand just how much social capital they can get from doing good reviews. Bacik said that the group was taking an overly simple view of the reviewing task; many developers hesitate to do reviews because they don't want to be seen as having missed something if a bug turns up. It was widely agreed that nobody should feel that way, since no one can be expected to catch everything. How to communicate that to the community as a whole is unclear, though. 

Torvalds said that a Reviewed-by tag mainly means that the reviewer will be copied on any bug reports; developers should add those tags liberally, he said. Gleixner added that maintainers make fools of themselves every other day. Hellwig said that he has been trying to review code outside of his comfort area; it takes a couple of times before he feels that he understands well enough to offer a Reviewed-by tag. Rostedt, though, raised the issue of bare Reviewed-by tags offered without any discussion, which can be a sign of somebody trying to game the system and get into the statistics. Bacik said that, if the maintainer does not know the reviewer, their Reviewed-by tag means nothing. 

Torvalds said that some subsystems are setting their requirements for contributors too high, making it hard for new developers to come in. Chinner added that the kernel's culture can be off-putting and not inclusive, making people fight to get their changes in. Bacik agreed, saying that there is no arbiter in the community; he said that Torvalds wants developers to figure things out for themselves, so disagreements over changes often end up as big battles. He would like to move to a system that is more encouraging of efforts to find solutions. 

Torvalds said that, while the community gets a lot of new contributors, it doesn't tend to get many new maintainers. The contributor and maintainer roles should be separated, he said. Chinner said that becoming a maintainer is often seen as a promotion for developers who do good work; Torvalds answered that people often see maintainers as some sort of "super developer", but they are really just managers. He took a moment to thank Konstantin Ryabitsev for the b4 tool, which has made life much easier for maintainers; the attendees responded with applause. 

As this part of the session came to a close, Williams said that part of the pay for reviewing work is autonomy within a subsystem, but that the community doesn't actually provide that autonomy. Instead, maintainers hold onto all of the decision power. Airlie answered that the DRM subsystem has done well with distributing that power among a number of developers. 

#### A support group

[![\[Steve Rostedt\]](https://static.lwn.net/images/conf/2023/lpc/SteveRostedt-sm.png)](<https://lwn.net/Articles/952038/>) A related session was run by Rostedt, who started by saying that he has heard a lot of maintainers and developers complaining about burnout. There are many things that could be done about this problem, but often all that a tired developer really needs is somebody to talk to. He is proposing the creation of a list of developers who are willing to lend an ear when the need arises. These developers would have no power, they would just be there to provide support and advice when a problem arises. 

Torvalds answered that if he wanted to talk to somebody, he wouldn't go to a kernel developer. Bacik, though, said that he is willing to do some basic support work. He can talk well with developers who are at the same level in the community, but his ability to get others to listen to him is not great. He suggested that Torvalds should take less of a laissez-faire approach to the development community and help solve problems more often. 

Chinner asked what problem Rostedt was trying to solve; Rostedt answered that many developers feel isolated and that they could benefit from a support group, but they don't know who to talk to. Bacik said that, with the developers he works with, he knows that problems can be worked out. But perhaps developers who lack that assurance could use some support. 

Williams asked whether the inability for developers to see each other for a couple of years contributed to problems; many people seemed to think that it did. 

At the end of this session, Torvalds said that about half of the emails he receives are private, rather than copied to the mailing lists. Developers are always welcome to send him a note when they are having problems; he has often had long discussions with developers about conflicts. Ts'o said that individual subsystems often have a decision maker who can bring conflicts to an end, resolving disputes by decree if they have to. The community lacks that resource for cross-subsystem issues, though. 

The next step will be for Rostedt to propose an addition to the kernel's process documentation describing this support group.
