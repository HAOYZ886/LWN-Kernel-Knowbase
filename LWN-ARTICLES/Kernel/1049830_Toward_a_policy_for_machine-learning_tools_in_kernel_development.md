---
title: "Toward a policy for machine-learning tools in kernel development"
url: https://lwn.net/Articles/1049830/
date: "December 11, 2025"
category: "Development tools-Large language models"
author: "By Jonathan Corbet December 11, 2025 Maintainers Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
December 11, 2025

* * *

[Maintainers Summit](<https://lwn.net/Articles/1049982/>)

The first topic of discussion at the 2025 Maintainers Summit has been in the air for a while: what role — if any — should machine-learning-based tools have in the kernel development process? While there has been a fair amount of controversy around these tools, and concerns remain, it seems that the kernel community, or at least its high-level maintainership, is comfortable with these tools becoming a significant part of the development process. 

Sasha Levin began the discussion by pointing to [a summary](<https://lwn.net/ml/all/aTYmE53i3FJ_lJH2@laps>) he had sent to the mailing lists a few days before. There is some consensus, he said, that human accountability for patches is critical, and that use of a large language model in the creation of a patch does not change that. Purely machine-generated patches, without human involvement, are not welcome. Maintainers must retain the authority to accept or reject machine-generated contributions as they see fit. And, he said, there is agreement that the use of tools should be disclosed in some manner. 

#### Just tools?

But, he asked the group: is there agreement in general that these tools are, in the end, just more tools? Steve Rostedt said that LLM-generated code may bring legal concerns that other tools do not raise, but Greg Kroah-Hartman answered that the current developers certificate of origin ("Signed-off-by") process should cover the legal side of things. Rostedt agreed that the submitter is ultimately on the hook for the code they contribute, but he wondered about the possibility of some court ruling that a given model violates copyright years after the kernel had accepted code it generated. That would create the need for a significant cleanup effort. 

[![\[Sasha Levin\]](https://static.lwn.net/images/conf/2025/ossjp/SashaLevin-sm.png)](<https://lwn.net/Articles/1049977/>) Ted Ts'o said that people worry about the copyright problems, but those same problems exist even in the absence of these tools. Developers could, for example, submit patches without going through the processes required by their employer — patches which, as a result, they have no right to submit. We do not worry about that problem now, he said, and it has almost never actually come up. Jiri Kosina said that these tools make code creation easy enough that the problem could become larger over time. Dave Airlie asked whether it makes sense to keep track of which models people are using. But, he said, any copyrighted code put into a patch by an LLM is likely to have come from the kernel itself. 

Levin mentioned that there had been some ethical concerns raised about LLM use and its effects on the rest of the world. Arnd Bergmann said that it could make sense to distinguish between which types of models are in use. Running one's own model locally is different from using a third party's tool. 

Linus Torvalds jumped in to say that he thought the conversation was overly focused on the use of LLMs to write code, but there has not, yet, been much of that happening for the kernel. So any problems around LLM-written code are purely hypothetical. But these tools _are_ being used for other purposes, including identifying CVE candidates and stable-backport candidates, and for [patch review](<https://lwn.net/Articles/1041694/>). Andrew Morton, Torvalds said, had recently shown an example of a machine-reviewed patch that was ""stunning""; it found all of the complaints that Torvalds had raised with the patch in question, and a few more as well. 

Alexei Starovoitov said that, within Meta, automated tools have been producing good reviews about 60% of the time, with another 20% having some good points. Less than 20% of the review comments have been false positives. Jens Axboe added that he has been testing with older patches and seeing similar results. He passed one five-line patch with a known problem to three human reviewers, and none of them found the bug. But the automated tool did find the problem (a reversed condition in a test); ""AI always catches that stuff"". 

Christian Brauner asked the group how many people use LLMs for coding; about four developers raised their hands. Shuah Khan expressed concern about access to LLMs; most of this work is being done behind corporate walls. Ts'o said that he has been using the review prompts posted by Chris Mason, originally written for Claude, with Gemini, with generally good results and at a relatively low cost. 

Torvalds, though, pointed out that developers have long been complaining about a lack of code review; LLMs may just solve that problem. They are not writing code at this point, he said, though that will likely happen at some point too. Once these systems start submitting kernel code, we will truly need automated systems to review all that code, he said. 

#### Proprietary systems

Konstantin Ryabitsev said that he had tried using some of these systems, but found them to be far too expensive; he also was worried about depending on proprietary technology. Brauner said that this usage had to be supported by employers, or perhaps the Linux Foundation could attempt to provide an automated review service. Ts'o said that the expense depends on how the system is used. One can pull in the entire kernel, using a lot of tokens; that will be expensive. The alternative is to create a set of review rules, reducing the token use by a factor of at least five. Khan repeated that not all developers will have equal access to this technology. 

Mark Brown was concerned about requiring submitters to run their patches through proprietary tools; some will surely object to that. Axboe suggested that the review tools should be run by subsystem maintainers, not submitters. 

I pointed out that, 20 years ago, the kernel community abruptly lost access to BitKeeper, highlighting the hazards of depending on proprietary tools. If the kernel community becomes dependent on these systems, development will suffer when the inevitable rug-pull happens. At some point, the cost of using LLMs will have to increase significantly if the companies behind them are to have a chance at reaching their revenue targets. 

Torvalds, though, called that concern a ""non-argument"". We do not have those tools today, he said; if they go away tomorrow, the community will just be back where it is now. Meanwhile, he said, we should take advantage of the technology industry's willingness to waste billions of dollars to get people to use these tools. Even if it only lasts a couple of years, it can help the community. 

Starovoitov said that he loves the reviews that the BPF community gets from the LLM systems. They ask good questions even when the reviews are wrong. Even better, developers respond to the questions, despite the fact that they are answering a bot; those answers can be used to help the models learn to do better in the future. But he acknowledged a recent three-day outage caused by some problems at GitHub; it ""felt devastating"". He was just waiting for the service to come back, since it does a better job of reviewing than he does. 

#### Disclosure

Levin shifted the discussion to disclosure requirements. There have been proposals for an Assisted-by tag that would name the specific tool used; should that tag be required for all tools, or just for LLMs? Torvalds said that he would like to see links to LLM-generated reviews, but that there is no need for a special review tag. Ts'o agreed, saying that people need to look at the reviews to determine whether they make sense, but he pointed out that a lot of reviews are not posted publicly. Starovoitov answered that the reviews in the BPF subsystem are posted as email responses to the patches. 

Kees Cook said that he didn't care about which specific tag is used, he just wants to know what he should use; Torvalds answered that there does not need to be a tag at all. The information could just be put into the changelog instead. Ryabitsev suggested putting it after the "`---`" marker so that it doesn't appear in the Git changelog, but Bergmann said he would prefer to have that information in the changelog. Torvalds accused the group of over-thinking the problem, saying that it was better to experiment and see what works. The community should encourage disclosure of tool use, but not make hard rules about how that disclosure should be done. 

Ts'o said that, in any case, it is not possible to count on submitters disclosing their tool use; some people may want to lie about it. Dan Williams said that disclosure rules would make it clear that the community values transparency in this area. Levin added that the nice thing about these tools is that they listen; if a disclosure rule is added to the documentation, the models will comply. Williams suggested a rule that all changelogs should mention leprechauns. 

As the session moved toward a close, Levin said he would post a documentation patch asking LLM tools to add an Assisted-by tag, but would not make an effort to enforce the rule. There was some final discussion on the details of that tag, which seems sure to evolve over time.
