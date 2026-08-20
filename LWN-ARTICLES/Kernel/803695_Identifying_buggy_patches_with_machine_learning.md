---
title: Identifying buggy patches with machine learning
url: https://lwn.net/Articles/803695/
date: "November 4, 2019"
category: "Development model-Stable tree"
author: "By Jonathan Corbet November 4, 2019 OSS EU"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 4, 2019

* * *

[OSS EU](<https://lwn.net/Archives/ConferenceByYear/#2019-Open_Source_Summit_Europe>)

The stable kernel releases are meant to contain as many important fixes as possible; to that end, the stable maintainers have been [making use of a machine-learning system](<https://lwn.net/Articles/764647/>) to identify patches that should be considered for a stable update. This exercise has had some success but, at the 2019 Open Source Summit Europe, Sasha Levin asked whether this process could be improved further. Might it be possible for a machine-learning system to identify patches that _create_ bugs and intercept them, so that the fixes never become necessary? 

Any kernel patch that fixes a bug, Levin began, should include a tag marking it for the stable updates. Relying on that tag turns out to miss a lot of important fixes, though. About 3-4% of the mainline patch stream was being marked, but the number of patches that _should_ be put into the stable releases is closer to 20% of the total. Rather than try to get developers to mark more patches, he developed his machine-learning system to identify fixes in the mainline patch stream automatically and queue them for manual review. 

This system uses a number of heuristics, he said. If the changelog contains language like "fixes" or "causes a panic", it's likely to be an important fix. Shorter patches tend to be candidates. Another indicator is the addition of code like: 

```
if (x == NULL)
            return -ESOMETHING;
```

In the end, it does turn out to be possible to automatically identify a number of fixes. But if that can be done, could it be possible to use a similar system to find bugs? That turns out to be a harder problem. Levin complained that nobody includes text like "this commit has a bug" or "this will crash your server" in their changelogs — a complaint that Andrew Morton [beat him to](<https://lwn.net/Articles/126795/>) by many years. Just looking at code constructs can only catch the simplest bugs, and there are already static-analysis systems using that technique. So he needed to look for something else. 

That "something else" turns out to be review and testing — or the lack thereof. A lot can be learned by looking at the reviews that patches get. Are there a lot of details in the review? Is there an indication that the [![\[Sasha Levin\]](https://static.lwn.net/images/conf/2019/osse/SashaLevin-sm.jpg)](<https://lwn.net/Articles/803699/>) reviewer actually tried the code? Does it go beyond typographic errors? Sentiment analysis can also be used to get a sense for how the reviewer felt about the patch in general. 

Not all reviewers are equal, so this system needs to qualify each reviewer. Over time, it is possible to conclude how likely it is that a patch reviewed by a given developer contains a bug. The role of the reviewer also matters; if the reviewer is a maintainer of — or frequent contributor to — the relevant subsystem, their review should carry more weight. 

A system can look at how long any given patch has been present in linux-next, how many iterations it has been through, and what the "quality" of the conversation around it was. Output from automated testing systems has a place, but only to an extent; KernelCI is a more reliable tester for ARM patches, but the 0day system is better for x86 patches. Manual testing tends to be a good indicator of patch quality; if a patch indicates that it has been tested on thousands of machines in somebody's data center for months, it is relatively unlikely to contain a bug. 

Then, one can also try to look at code quality, but that is hard to quantify. Looking at the number of problems found in the original posting of a patch might offer some useful information. But Levin is unsure about how much can be achieved in this area. 

Once the data of interest has been identified, it is necessary to create a training set for the system. That is made up of a big pile of patches, of course, along with a flag saying whether each contains a bug or not. The `Fixes` tags in patches can help here, but not all bugs really matter for the purposes of this system; spelling fixes or theoretical races are not the sort of problem he is looking for. In the end, he took a simple approach, training the system on patches that were later reverted or which have a `Fixes` tag pointing to them. 

That led to some interesting information about where and when bugs are introduced. He had thought that bugs would generally be added during the merge window, then fixed in the later -rc releases, but that turned out to be wrong. On a lines-of-code basis, a patch merged for one of the late -rc releases is three times more likely to introduce a bug than a merge-window patch. 

Patches queued for the merge window, it seems, are relatively well tested. Those added late in the cycle, instead, are there to fix some other problem and generally get much less testing — or none at all. Levin said that things shouldn't be this way. There is no reason to rush fixes late in the development cycle; nobody runs mainline kernels in production anyway, so it is better to give those patches more testing then push them into the stable updates when they are really ready. Developers should, he said, trust the system more and engage in less "late-rc wild-west stuff". 

Levin complained to Linus Torvalds about this dynamic; Torvalds agreed with the explanation but said that the system was designed that way. Late-cycle problems tend to be more complex, so the fixes will also be more complex and more likely to create a new bug. Levin agreed that this is the case, but disagreed with the conclusion; he thinks that the community should be more strict with late-cycle patches. 

Back to the machine-learning system, he said that he is currently using it to flag patches that need more careful review; that has enabled him to find a number of bugs in fixes that were destined for stable updates. Parts of this system are also being used to qualify patches for the stable releases. The goal of detecting buggy patches in general still seems rather distant, though. 

Levin concluded with some thoughts on improving the kernel development process. The late-rc problem needs to be addressed; we know there is a problem there, he said, so we should do something about it. Testing of kernel patches needs to be improved across the board; the testing capability we have now is rather limited. More testing needs to happen on actual hardware to be truly useful. He would also like to see some standardization in the policy for the acceptance of patches, including how long they should be in linux-next, the signoffs and reviews needed, etc. These policies currently vary widely from one subsystem to the next, and some maintainers seem to not care much at all. That, he said, is not ideal and needs to be improved. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting your editor's travel to the event.]
