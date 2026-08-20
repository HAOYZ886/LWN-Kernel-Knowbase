---
title: Large language models for patch review
url: https://lwn.net/Articles/1041694/
date: "October 16, 2025"
category: "Development tools-Large language models"
author: "By Jonathan Corbet October 16, 2025"
---

By **Jonathan Corbet**  
October 16, 2025

There have been many discussions in the free-software community about the role of large language models (LLMs) in software development. For the most part, though, those conversations have focused on whether projects should be accepting code output by those models, and under what conditions. But there are other ways in which these systems might participate in the development process. Chris Mason recently [started a discussion](<https://lwn.net/ml/all/fc05f97b-1257-4dee-966f-ba66fff8aef1@meta.com>) on the Kernel Summit discussion list about how these models can be used to review patches, rather than create them. 

Mason's focus was on how LLMs might reduce the load on kernel maintainers by catching errors before they hit the mailing lists, and by helping contributors increase the quality of their submissions. To that end, he has put together a [set of prompts](<https://github.com/masoncl/review-prompts>) that will produce reviews in a format that maintainers are used to: ""The reviews are meant to look like emails on lkml, and even when wildly wrong they definitely succeed there"". He included a long list of sample reviews, some of which hit the mark and others of which did not. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The prompts are interesting in their own right; they can be seen as constituting the sort of comprehensive patch-review documentation that nobody ever quite got around to writing for humans to use. Perhaps that reflects a higher level of confidence that the LLM will actually _read_ all of this material. These prompts add up to thousands of lines of material, starting with [core guidance](<https://github.com/masoncl/review-prompts/blob/main/review-core.md>) like: 

> Struct changes → verify all users use the new struct correctly 
> 
> Public API changes → verify documentation updates [...] 
> 
> **Tone Requirements** : 
> 
>   * **Conversational** : Target kernel experts, not beginners 
>   * **Factual** : No drama, just technical observations 
>   * **Questions** : Frame as questions about the code, not accusations 

Most of the prompts consist of guidance specific to subsystems like [locking](<https://github.com/masoncl/review-prompts/blob/main/locking.md>) (""You're not smart enough to understand smp_mb(), smp_rmb(), or smp_wmb() bugs yet"") and [networking](<https://github.com/masoncl/review-prompts/blob/main/networking.md>) (""Socket can outlive its file descriptor""). All told, it resembles the sort of rule collection one saw in the expert systems that [were going to take over the world](<https://en.wikipedia.org/wiki/Fifth_Generation_Computer_Systems>) in the 1980s. As noted in [the README file](<https://github.com/masoncl/review-prompts/blob/main/README.md>), ""the false positive rate is pretty high right now, at ~50%"", so there is still some room for improvement. 

In the ensuing discussion, nobody seemed to think that using LLMs in this way was a bad idea. Sasha Levin [called it](<https://lwn.net/ml/all/aOaujzH1dl-OEbsO@laps>) ""a really great subject to discuss"", and said that, in the [previous discussions](<https://lwn.net/Articles/1032612/>) on LLM use by kernel developers, the concerns that were raised about LLMs drowned out out any attempt to find the places where they could be useful. Paul McKenney [remarked](<https://lwn.net/ml/all/854ab19d-7bba-48ac-b822-77b72e84bee2@paulmck-laptop>) that using this technology to review code written by others ""seems much safer than using it to generate actual code"". Krzysztof Kozlowski [noted](<https://lwn.net/ml/all/be5094b9-fb20-462e-ad2f-2b58e520b949@kernel.org>) that Qualcomm has created a similar system and made it available. 

There were some concerns raised about the proprietary nature of these systems; Konstantin Ryabitsev was just one of a few who [drew parallels](<https://lwn.net/ml/all/20251008-lively-vermilion-snail-beff9a@lemur>) with [the BitKeeper experience](<https://lwn.net/Articles/130746/>) that (briefly) brought kernel development to a halt just over 20 years ago. Laurent Pinchart [stated clearly](<https://lwn.net/ml/all/20251009091405.GD12674@pendragon.ideasonboard.com>) that there are limits to how much proprietary tools can be used or required: 

> Forcing contributors to pay for access to proprietary tools is not acceptable. Forcing contributors to even run proprietary tools is not acceptable. 

He also expressed concerns that the companies behind LLMs would make them available to developers for free to encourage adoption — until the community is well locked in, at which point access could quickly become expensive. Mason, though, [was unworried](<https://lwn.net/ml/all/d4f98276-ab5d-43ca-9662-017420154e4a@meta.com>) about lock-in, saying that the prompts are sufficiently generic to be adaptable to any system. James Bottomley [suggested](<https://lwn.net/ml/all/3996fd684c497c7bcb4ad406ff3cec99df7180df.camel@HansenPartnership.com>) that LLMs would not be proprietary forever, but Pinchart [argued](<https://lwn.net/ml/all/20251010115334.GB28598@pendragon.ideasonboard.com>) against relying on proprietary software in the hope that there will eventually be free alternatives. 

There was some disagreement over who an LLM-based review tool should be created for. Mason's target was maintainers, but Andrew Lunn [argued](<https://lwn.net/ml/all/fe0b8ef3-6dc7-4220-842b-0d5652cae673@lunn.ch>) that the plan should be for developers to run these tools themselves before posting code for review. That, he said, would further reduce the workload on maintainers, who would only need to run LLM review to verify the the submitter had already done so. 

Pinchart, along with others, [pointed out](<https://lwn.net/ml/all/20251008192934.GH16422@pendragon.ideasonboard.com>) that getting developers to use the tools (such as `checkpatch.pl`) that exist now is difficult; he wondered how submitters could be encouraged to run any new tools. Tim Bird [suggested](<https://lwn.net/ml/all/MW5PR13MB5632D8B5B656E1552B21159FFDE1A@MW5PR13MB5632.namprd13.prod.outlook.com>) annotating patches with a list of the tools that have been run on them so that maintainers could see that history. Bottomley, instead, [said](<https://lwn.net/ml/all/d08330417052c87b58b4a9edd4c0e8602e4061f2.camel@HansenPartnership.com>) that these tools should be run automatically on patches sent to the mailing lists, much like the checks that the [0day robot](<https://www.intel.com/content/www/us/en/developer/topic-technology/open/linux-kernel-performance/overview.html>) runs on posted patches now. Bird, though, [said](<https://lwn.net/ml/all/MW5PR13MB5632FC46AD54998C4C584F91FDE1A@MW5PR13MB5632.namprd13.prod.outlook.com>) that running the tools should be expected of submitters. ""It then becomes a cost for the contributor instead of the upstream community, which is going to scale better."" 

Mason [was clear](<https://lwn.net/ml/all/72b9b81c-765b-4047-bb3b-40b2a8a6e563@meta.com>) in his belief that LLM-generated reviews should happen in public as part of the submission process: 

> I think it's also important to remember that AI is sometimes wildly wrong. Having the reviews show up on a list where more established developers can call bullshit definitely helps protect against wasting people's time. 

Linus Torvalds, in [his one contribution to the discussion](<https://lwn.net/ml/all/CAHk-=wj3fQVEcAqy82JnrX2KKi4NjnEGGSH2Pf_ztnLCcveWkQ@mail.gmail.com>), agreed. He was about the only one to express concerns about the technology, saying ""I think we've all seen the garbage end of AI, and how it can generate more work rather than less"". Mason [agreed](<https://lwn.net/ml/all/c61d69f9-3434-44f7-8379-5d4aa280780e@meta.com>) that Torvalds's concerns were relevant, based on his own experience: 

> My first prompts told AI to assume the patches had bugs, and it would consistently just invent bugs. That's not the end of the world, but the explanations are always convincing enough that you'd waste a bunch of time tracking it down. 

Torvalds mentioned the [scraper problem](<https://lwn.net/Articles/1008897/>) as well. His concerns notwithstanding, he believes that this technology will prove helpful, but he feels that its initial adoption has to be aimed at making life easier for maintainers. ""So I think that only once any AI tools are actively helping maintainers in a day-to-day workflow should people even *look* at having non-maintainers use them"". 

The conversation wound down shortly after that. One clear conclusion, though, is that these tools seem destined to play an increasing role in the kernel-development process. At some point, we will likely start seeing machine-generated reviews showing up on the mailing lists; then, perhaps, the real value of LLM-based patch-review tools will start to become clear. It will be interesting to see how the inevitable related discussion at the [2025 Maintainer Summit](<https://events.linuxfoundation.org/linux-kernel-maintainer-summit/>) in December plays out.
