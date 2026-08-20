---
title: "The memory-management subsystem development process"
url: https://lwn.net/Articles/787705/
date: "May 7, 2019"
category: "Development model-Code review"
author: "By Jonathan Corbet May 7, 2019 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 7, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

One fixture of the memory-management track at the Linux Storage, Filesystem, and Memory-Management Summit is a discussion with subsystem maintainer Andrew Morton on how the development process is going. The 2019 version indicated that the memory-management developers are mostly happy with how the process is working, but there are still things that they would like to see changed. While some of the issues are old and intractable, others may be amenable to short-term improvement. 

Morton started by reminding the group that the [2018 development-process session](<https://lwn.net/Articles/752985/>) had established the goal that all memory-management patches would be reviewed before they go upstream. That goal, he said, has been "easily achieved". That said, while many of the reviews are good, some of them are less so. But, he asked, did the group find the reviews as a whole useful? The answer was a clear "yes". Aneesh Kumar said that insisting on reviews puts pressure on developers and helps to make patch review actually happen; Morton answered with a hope that, as the process becomes better understood, managers will allow their developers more time for patch review. 

At this point, Mel Gorman went into a long soliloquy about patch review which, he said, is not always achieving its objectives in the memory-management subsystem. For example, developers will post complex [![\[Andrew Morton\]](https://static.lwn.net/images/conf/2019/lsfmm/AndrewMorton-sm.jpg)](<https://lwn.net/Articles/787733/>) patch sets without any justification and have that pointed out in review; then the next version turns out to be even more complex. Memory management is not a black-and-white field, there are consequences to everything and often little supporting data is provided to describe the consequences of specific patches. 

At the same time, Gorman continued, patches that are seen as important fixes sometimes are rejected, usually because of a single objector blocking progress. In other subsystems, the maintainer just makes a call on patches like that but, in memory management, they can get stuck indefinitely. He mentioned the removal of the legacy block layer, which took a few rounds. There were objections, which were addressed; eventually the maintainer just merged it as the right thing to do. 

Things seem to happen differently in memory-management. For example, Gorman said, the reversion of Andrea Arcangeli's patch fixing performance problems with transparent huge page allocation should have been overridden. (That episode was [discussed](<https://lwn.net/Articles/787434/>) earlier in the day). The fix was reverted as the result of an objection from "a single developer with an unreproducible workload"; sometimes, he said, you just have to push work through. If you wait for the controversy to end, you'll never make progress, he concluded. 

Michal Hocko raised a perennial problem in memory management (and beyond): there are still more people writing patches than reading them. Johannes Weiner asked if there were quality problems as a result; Hocko replied in the affirmative, saying that some parts of the subsystem are more problematic than others. Gorman said that memory-management is far from alone in suffering from this problem; the group should concede that there are quality issues, he said, but nobody should think that the problem is unique to this subsystem. Morton asked the developers to bring his attention to problems when they come up; Gorman answered those times when he appears on the list, quickly fixes something, then disappears are a good sign that something has gone wrong. 

In the traditional kernel maintainer model, Morton said, the maintainer tends to get involved late in the development of a patch set, when it is nearly completed. The maintainer's job at that point is to make sure that it get upstream; things tend not to change after a patch is applied to the subsystem maintainer's repository. Morton doesn't work that way, though; he gets involved earlier and tries to help developers get their work into shape. His ‑mm tree is a part of the development process, not its culmination. Is this good or bad? 

Vlastimil Babka said that developers often don't understand that ‑mm works that way; they think that once something appears there, the work is done. Morton answered that he has gotten "less promiscuous" lately, passing over the first version of many patch sets. Hocko said that, as a reviewer, he can lose the big picture when followup fixes are added to ‑mm (a topic that came up in 2018 as well). That is, he said, a huge problem that has led to incorrect merges in the past. That is a direct result of the early merging strategy; he would rather see developers just send a new series when things need to be fixed. 

Gorman said that the early merging approach has both merits and limits. It makes it hard for developers to work against linux-next, which is made more volatile by the inclusion of unready code. Gorman's own patches tend to come with a lot of supporting data, but all of those results are measured with mainline kernels; the numbers can change significantly in ‑mm, forcing him to reverify things. It would be nice, he said, if ‑mm had a proper set of topic branches for developers to work against; that sentiment was echoed by others in the room as well. Hocko said that different parts of the memory-management subsystem need different base trees for development; ‑mm is a single point of failure now. Other subsystems are trying to add redundancy to the maintenance process; he would like to see something similar happen for memory management. 

Morton said that, when developers work against the mainline tree, things usually just work, and he is comfortable with that. Gorman replied, though, that it is hard to know which patches in ‑mm will go upstream anytime soon. That could be improved with a set of standardized branches in ‑mm, so that developers can have a better idea of what the next merge window will look like. 

As the session ran out of time and dinner beckoned, Babka noted that most maintainers put detailed summaries into their pull requests; that doesn't happen with ‑mm. When asked, I could only agree; a pull request that reads "most of ‑mm" is not particularly helpful. Morton closed the session by saying that he would try to do better.
