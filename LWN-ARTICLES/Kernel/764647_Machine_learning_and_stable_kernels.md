---
title: Machine learning and stable kernels
url: https://lwn.net/Articles/764647/
date: "September 12, 2018"
category: "Development model-Stable tree"
author: "By Jake Edge September 12, 2018 OSS NA"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
September 12, 2018

* * *

[OSS NA](<https://lwn.net/Archives/ConferenceByYear/#2018-Open_Source_Summit_North_America>)

There are ways to get fixes into the stable kernel trees, but they require humans to identify which patches should go there. Sasha Levin and Julia Lawall have taken a different approach: use machine learning to distinguish patches that fix bugs from others. That way, all bug-fix patches could potentially make their way into the stable kernels. Levin and Lawall gave a talk describing their work at the [2018 Open Source Summit North America](<https://events.linuxfoundation.org/events/open-source-summit-north-america-2018/>) in Vancouver, Canada. 

Levin began with a quick introduction to the stable tree and how patches get into it. When a developer fixes a bug in a patch they can add a "stable tag" to the commit or send a mail to the [stable mailing list](<http://vger.kernel.org/vger-lists.html#stable>); Greg Kroah-Hartman will then pick up the fix, evaluate it, and add it to the stable tree. But that means that the stable tree is only getting the fixes that are pointed out to the stable maintainers. No one has time to check all of the commits to the kernel for bug fixes but, in an ideal world, all of the bug fixes would go into the stable kernels. Missing out on some fixes means that the stable trees will have more security vulnerabilities because the fixes often close those holes—even if the fixer doesn't realize it. 

[ ![\[Sasha Levin\]](https://static.lwn.net/images/2018/ossna-levin-sm.jpg) ](<https://lwn.net/Articles/764650/>)

But the stable tags are not that effective as a way to communicate bug fixes, Levin said. Patch authors are often unsure whether a patch is stable material and sometimes a patch is not stable material when it is written, but is at the time it gets merged. Beyond that, various subsystems have different rules about marking patches for stable; the USB subsystem expects authors to add the stable tag, while, in the networking subsystem, stable tags are all added by David Miller. Levin estimates that only half of the patches that fix bugs actually get marked for the stable kernels. 

"Hey, we have that Greg guy, let's make him look at all the commits", he said with a chuckle. In truth, that is not even remotely possible. It takes at least a minute or two to look at a patch and see if it is relevant for stable. There are so many commits, all over the kernel, that jumping around to various subsystems looking at them all is mentally exhausting. With 14,000 commits per release, even a minute and a half per patch would result in 350 hours of patch review. When Levin does patch review for the stable tree, he limits himself to doing so an hour at a time, otherwise he "would go crazy". 

One obvious solution would be to automate the process somehow. In order to do that, you need to figure out what indicates that a patch might be a bug fix. There is no big flag, instead there are many small things that are indicative of the right kind of patch. There are often hints in the commit message or code constructs (e.g. adding a `spin_unlock()`) that would lead one toward the "fix" classification. There are also clues in the subsystem that the patch applies to and the author of the patch; some subsystems are in maintenance mode, so they mostly only get fixes, while some authors tend toward bug fixing. 

He gave some examples. A commit message that has strings like "bug", "fixes", or "memory leak" are prime candidates. Adding unbalanced locking, a test for null, or an additional branch or return would be indicators as well. Certain bots, such as syzbot or the 0day bot, are likely to have reported bugs that need fixing; he has seen lots of commits that reference syzbot, but not stable, he said. 

#### Building a neural net

So he decided to build a neural network to see what it could do to classify patches. He started out "not knowing anything", so he took a somewhat naive approach. He took the most common 10,000 words in kernel commit messages and tagged which of those were in a candidate patch. That was one of the inputs; others were information on code metrics for the changes in the commit, author information (in the hopes that it matters somewhat, he said), the involved parties (reviewers, committers), and which files were modified. Some files tend to have more fixes than others; files with hardware quirks or PCI IDs are trivial examples of that. 

He then trained the neural network with commits from Linux 3.0 to 4.16, where the "true" value was whether the commit was in the stable tree. He started off training it on his laptop, but that took too long, so he switched to a "beefy GPU" provided by his employer, Microsoft. It took about a month to train the model down to less than a 5% error rate. That cost about $2000 for the month. 

The results have been good; it is far easier to look at 1000 commits, rather than 14,000. The output from the neural network has led to more than 3000 commits to various stable trees, many of which were critical fixes. Many of those do not have CVE numbers associated with them, Levin said, because he believes that people only get CVEs for publicity purposes. The number of rejections and reversions among the patches identified by the system is comparable to that of the commits tagged by humans for stable. 

There are some imperfections in the system. It can identify bug fixes, but it cannot determine whether the fix is relevant to a particular kernel version. In addition, there are lots of fixes that didn't go into the stable tree—part of the reason behind this project—which means that the training data is not perfect. Beyond that, there is no universal definition of what a "bug" is; people can look at the same "fix" and disagree on whether it even _is_ a bug fix. 

#### Neural network background

With that, Lawall took over to try to fill in some of the background of neural networks to explain what Levin has done and how these networks apply to the problem of bug-fix identification. She is not a neural network expert, she said, but has been looking into the topic of late. Lawall also reported on the results of some of the related research she has been doing recently. 

[ ![\[Julia Lawall\]](https://static.lwn.net/images/2018/ossna-lawall-sm.jpg) ](<https://lwn.net/Articles/764651/>)

She started, like Levin, by identifying the features of a patch that might lead a human to suspect they fix a bug. That includes commit message clues, developers involved, and the code changes made. Once these things have been identified, though, some kind of weight needs to be assigned to them. For example, having "bug" or "fix" in the commit message could be given a weight of 0.3, the commit coming from a well-known developer might be given a weight of 0.2, and a locking or null-test change might get a weight of 0.4. Those weights would be applied to values that score the commit's characteristics based on the criteria (so, for example, "well-known" could have a score that ranked developers). The weights are multiplied by the score and added up to give a confidence value that a commit fixes a bug; choosing a cut-off for the confidence value results in a yes or a no answer for a given patch. 

But the weighting values are arbitrary; should "bug" or "fix" in the commit message be 0.28 or 0.1, perhaps? The original 0.3 value was set based on intuition, not rigorous testing. In addition, well-known developers do a lot of different things, not just fixes; should the calculation really be taking both of those into account? Maybe it is only if a well-known developer made a commit with "fix" in it that should be scored highly. 

So there is a need to optimize the weighting values and to take combinations of features into account. That is where a [feedforward neural network](<https://en.wikipedia.org/wiki/Feedforward_neural_network>) can come into play. These networks are organized with three layers: input, hidden, and output. The layer organization is a way of describing the formula for making the decision; there are weights associated with each of the steps in the paths through the network. But where do _those_ weights come from? 

The weights come from the training process. Data that has results with expected values can be used to "back-propagate" weights in order to tune the model. Training is a process of "moving in some direction in our weight space" to produce better results, Lawall said. It is a [hill-climbing problem](<https://en.wikipedia.org/wiki/Hill_climbing>) where each iteration tries to improve on the last. At some point, she said, you decide that the error is small enough and stop. 

#### Improvements?

That is what Levin has done; she wondered what could be done to improve on that. The features that Levin chose were fairly arbitrary: maybe the author is important, but maybe not. Perhaps there are other features of the patches that are not being used but would produce better results. In addition, how can the neural network be trained to reason about code? The code features that Levin uses are coarse-grained, but fine-grained code features may be too costly to work with. 

She and her co-researchers have been looking at [convolutional neural networks](<https://en.wikipedia.org/wiki/Convolutional_neural_network>) (CNNs), which have traditionally been used in image processing. More recently, they have been used for natural language processing. In order to use them on code, though, the important features of code need to be identified. There is a temptation to add more and more features in order to ensure that the right ones are present, but each feature adds to the search space, thus to the cost of training and optimizing the network. 

A CNN has the concept of a filter that can be applied to the data to find where it matches best. You can start with a random filter or one based on intuition about the features of interest; then the training optimizes those filters. That can lead to a smaller set of features due to filters being discarded or combined in the process. 

For a patch, there are really two pieces: the commit message and the actual code changes. The commit message is English text, so strategies for processing natural language can be applied. Code is trickier; the representation of code changes needs to be worked out. She gave an example change: 

```
-       if (cpuidle_sysfs_monitor.hw_states_num == 0)
        +       if (cpuidle_sysfs_monitor.hw_states_num <= 0)
                        return NULL;
```

In that change, there is only one character different (`<` instead of `=`) but that may not be the right level to look at it. The `==` operator has changed to `<=`, which is a token-level change. That may still be too low-level for distinguishing fixes. Other options would be the whole "atomic statement" (the `if` line) or the full statement (the `if` and the `return`). They decided to go with the atomic statement option. 

Once that was decided, there is still more to determine. How do you break down the `if` line in a way that allows the model to generalize changes of this sort? What specific pieces of the line should be retained and which should be generalized? For the most part, they have decided to keep the C language pieces (e.g. `if`, `(`, `.`, `==`), while generalizing the identifiers and value (so `hw_states_num` and `cpuidle_sysfs_monitor` are simply identifiers, while `0` is an integer). Those may not be the best choices, but that is what they are working with currently. Representing commits, with multiple hunks of added and removed lines, will require multiple neural networks, she said; that is a work in progress. 

The resulting neural network is called PatchNet. They did some comparative testing between PatchNet, Levin's method, and a simple keyword check for "bug" or "fix" in the commit message. They took around 80K commits from Linux 3.0 to 4.12 that were roughly balanced between stable and non-stable commits. Levin took their data set and trained it using his methods. They then calculated two metrics for each. "Precision" measures the percentage of patches classified as stable by the model or test that were actually in the stable kernels. "Recall" measures the percentage of patches in the stable kernels that were classified as bug fixes by the three systems. 

The results were interesting, but somewhat inconclusive at this point. As might be guessed, the simple bug/fix test did not fare as well as either of the other two. For precision, Levin's model and PatchNet were essentially equal at around 85%. For recall, however, PatchNet came it at around 90%, while Levin's model was about 80%. PatchNet is better than Levin's technique at recognizing the full-breadth of patches in the stable tree, though it is far from clear to me how that might be used in the future. 

There are lots of other things that can be tried, Lawall said; it is early in the exploration of this technique. She and her team have made various choices, some of which may be good or bad; some experimentation with different choices seems warranted. The weighting in a neural network is reputed to be hard to understand, but it might be worth trying to peer inside these models to see if anything useful can be gleaned to improve their results. Other areas for fruitful research might be choosing which stable versions a patch applies to or to identify bug-introducing patches. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to attend Open Source Summit in Vancouver.]
