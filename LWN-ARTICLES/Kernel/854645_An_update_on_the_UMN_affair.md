---
title: An update on the UMN affair
url: https://lwn.net/Articles/854645/
date: "April 29, 2021"
category: "Security-Patch verification"
author: "By Jonathan Corbet April 29, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 29, 2021

On April 20, the world became aware of [a research program](<https://lwn.net/Articles/853717/>) conducted out of the University of Minnesota (UMN) that involved submitting intentionally buggy patches for inclusion into the Linux kernel. Since then, a paper resulting from this work has been withdrawn, various letters have gone back and forth, and numerous patches from UMN have been audited. It's clearly time for an update on the situation. 

The writing of [a paper on this research [PDF]](<https://github.com/QiushiWu/QiushiWu.github.io/blob/main/papers/OpenSourceInsecurity.pdf>) was not the immediate cause of the recent events; instead, it was the posting of a buggy patch originating from an experimental static-analysis tool run by another developer at UMN. That led developers in the kernel community to suspect that the effort to submit intentionally malicious patches was still ongoing. Since then, it has become apparent that this is not the case, but by the time the full story became clear, the discussion was already running at full speed. 

The old saying still holds true: one should not attribute to malice that which can be adequately explained by incompetence. 

On April 22, [a brief statement](<https://lwn.net/Articles/854064/>) was issued by the Linux Foundation technical advisory board (or TAB, of which your editor is a member) stating that, among other things, the recent patches appeared to have been submitted in good faith. Meanwhile, the Linux Foundation and the TAB sent a letter to the UMN researchers outlining how the situation should be addressed; that letter has not been publicly posted, but [ZDNet apparently got a copy](<https://www.zdnet.com/article/the-linux-foundations-demands-to-the-university-of-minnesota-for-its-bad-linux-patches/>) from somewhere. Among other things, the letter asked for a complete disclosure of the buggy patches sent as part of the UMN project and the withdrawal of the paper resulting from this work. 

In response, the UMN researchers posted [an open letter](<https://lwn.net/ml/linux-kernel/CAK8KejpUVLxmqp026JY7x5GzHU2YJLPU8SzTZUNXU2OXC70ZQQ@mail.gmail.com/>) apologizing to the community, followed a few days later by [a summary of the work they did [PDF]](<https://www-users.cs.umn.edu/%7Ekjlu/papers/full-disclosure.pdf>) as part of the "hypocrite commits" project. Five patches were submitted overall from two sock-puppet accounts, but [one of those](<https://lwn.net/ml/linux-kernel/20200804183650.4024-1-jameslouisebond@gmail.com/>) was an ordinary bug fix that was sent from the wrong account by mistake. Of the remaining four, [one of them](<https://lwn.net/ml/linux-kernel/20200821031209.21279-1-acostag.ubuntu@gmail.com/#t>) was an attempt to insert a bug that was, itself, buggy, so the patch was actually valid; the other three ([1](<https://lwn.net/ml/linux-kernel/20200809221453.10235-1-jameslouisebond@gmail.com/>), [2](<https://lwn.net/ml/linux-kernel/20200821034458.22472-1-acostag.ubuntu@gmail.com/>), [3](<https://lwn.net/ml/linux-kernel/20200821070537.30317-1-jameslouisebond%40gmail.com/>)) contained real bugs. None of those three were accepted by maintainers, though the reasons for rejection were not always the bugs in question. 

The paper itself has been withdrawn and will not be presented in May as was planned. One can, hopefully, assume that UMN will not be pursuing similar lines of research anytime soon. 

#### Patch re-review

One immediate result of the attention drawn to UMN's activities was a loss of trust in its developers, combined with a desire in some quarters to take some sort of punitive action. Thus, one of the first things that happened when this whole affair exploded was the posting by Greg Kroah-Hartman of [a 190-part patch series](<https://lwn.net/ml/linux-kernel/20210421130105.1226686-1-gregkh@linuxfoundation.org/>) reverting as many patches from UMN as he could find. Actually, it wasn't all of them; he mentioned a list of 68 others requiring manual review because they do not revert easily. 

As it happens, these ""easy reverts"" also needed manual review; once the initial anger passed there was little desire to revert patches that were not actually buggy. That review process has been ongoing over the course of the last week and has involved the efforts of a number of developers. Most of the suspect patches have turned out to be acceptable, if not great, and have been removed from the revert list; if your editor's count is correct, 42 patches are still set to be pulled out of the kernel. 

For those 42 patches, the reasoning behind the revert varies from one to the next. In some cases, the patches apply to old and presumably unused drivers and nobody can be bothered to properly review them. In others, the intended change was done poorly and will be reimplemented in a better way. And some of the patches contained serious errors; these definitely needed to be reverted (and should not have been accepted in the first place). 

A look at the full set of UMN patches reinforces some early impressions, though. First is that almost all of them do address some sort of real (if obscure and hard to hit) problem; there was a justification for writing a patch. While many of these fixes showed a low level of understanding of what the code was doing and thus contained errors, it seems unlikely that any of them were malicious in their intent. 

That said, there are multiple definitions of "malice". To some of the developers involved, posting unverified patches from an experimental static-analysis tool without disclosing their nature is a malicious act. It is another form of experiment involving non-consenting humans. At a minimum, it is a violation of the trust that is required for the kernel's development community to work effectively. 

The 42 bad patches out of 190 is a 22% bad-patch rate. Chances are, a detailed review of 190 patches from almost any kernel developer would turn up a few that, in retrospect, were not a good idea. Hopefully that rate would not approach 22%, though. But it must be said that all of those patches were accepted by subsystem maintainers throughout the kernel, which is not a great result. Perhaps that is a more interesting outcome than the one that the original "hypocrite commit" researchers were looking for. They failed in their effort to deliberately insert bugs, but were able to inadvertently add dozens of them. 

Meanwhile, there is still the list of patches that did not revert cleanly. That list has not been posted publicly, but Kroah-Hartman did start with [a subset of seven of them](<https://lwn.net/ml/linux-kernel/20210429130811.3353369-1-gregkh@linuxfoundation.org/>). He also noted that the TAB will be publishing a full report of the audit of all these patches once it is complete. Thus far, none of these patches have actually been reverted in the mainline; that seems likely to happen toward the end of the 5.13 merge window. 

#### Lessons learned

One of the key lessons from this series of events would clearly be: do not use a free-software development community as a sort of free validation service for your experimental tool. Kernel developers are happy to see new tools created and — if the tools give good results — use them. They will also help with the testing of those tools, but they are less pleased to be recipients of tool-inspired patches that lack proper review and an explanation of what is going on. 

Another lesson is something we already knew: kernel maintainers (and maintainers of many other free-software projects) are overworked and do not have the time to properly review every patch that passes through their hands. They are, as a result, forced to rely on the trustworthiness of the developers who submit patches to them. The kernel development process is, arguably, barely sustainable when that trust is well placed; it will not hold together if incoming patches cannot, in general, be trusted. 

The corollary — also something we already knew — is that code going into the kernel is often not as well reviewed as we like to think. It is comforting to believe that every line of code merged has been carefully vetted by top-quality kernel developers. Some code does indeed receive that kind of review, but not all of it. Consider, for example, the 5.12 development cycle (a relatively small one), which added over 500,000 lines of code to the kernel over a period of ten weeks. The resources required to carefully review 500,000 lines of code would be immense, so many of those lines, unfortunately, received little more than a cursory looking-over before being merged. 

One final lesson that one might be tempted to take is that the kernel is running a terrible risk of malicious patches inserted by actors with rather more skill and resources than the UMN researchers have shown. That could be, but the simple truth of the matter is that regular kernel developers continue to insert bugs at such a rate that there should be little need for malicious actors to add more. The 5.11 kernel, released in February, has accumulated 2,281 fixes in stable updates through 5.11.17. If one makes the (overly simplistic) assumption that each fix corrects one original 5.11 patch, then 16% of the patches that went into 5.11 have turned out (so far) to be buggy. That is not much better than the rate for the UMN patches. 

So perhaps that's the real lesson to take from this whole experience: the speed of the kernel process is one of its best attributes, and we all depend on it to get features as quickly as possible. But that pace may be incompatible with serious patch review and low numbers of bugs overall. For a while, we might see things slow down a little bit as maintainers feel the need to more closely scrutinize changes, especially those coming from new developers. But if we cannot institutionalize a more careful process, we will continue to see a lot of bugs and it will not really matter whether they were inserted intentionally or not.
