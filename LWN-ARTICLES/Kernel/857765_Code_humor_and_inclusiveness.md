---
title: Code humor and inclusiveness
url: https://lwn.net/Articles/857765/
date: "June 11, 2021"
category: "Development model-Diversity"
author: "By Jonathan Corbet June 11, 2021"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 11, 2021

Free-software development is meant to be fun, at least some of the time. Even developers of database-management systems seem to think that it is fun; there is no accounting for taste, it seems. Part of having fun is certainly allowing the occasional exercise of one's sense of humor while working on the code. But, as some recent "fix" attempts show, humor does not always carry through to developers all over the planet. Balancing humor and inclusiveness is always going to be a challenge for our community. 

There is [a function in the kernel scheduler](<https://elixir.bootlin.com/linux/v5.12.8/source/kernel/sched/topology.c#L1927>) that will, if things go badly wrong, inform users that a bug is present, describing it as ""arch topology borken"". The message is, of course, breathtaking in its clarity; users will immediately know what to do when they see it. Recently, though, a well-intentioned developer [sent a patch](<https://lwn.net/ml/linux-kernel/20210526021856.9060-1-thunder.leizhen@huawei.com/>) changing the message to read "broken" instead. This patch was not accepted, but it did result in a brief discussion explaining the nature of the intended joke to the submitter. 

The following day, somebody else attempted to fix [another scheduler function](<https://elixir.bootlin.com/linux/v5.12.8/source/kernel/sched/core.c#L7636>) containing a comment that cites ""histerical raisins"". This time, the poster was [instructed](<https://lwn.net/ml/linux-kernel/YK+5eKc4kx4+ALD1@hirez.programming.kicks-ass.net/>) to search for "humour", which is rather less illuminating. Other examples abound; one of the more amusing such attempts came last year, when a developer [concluded](<https://lwn.net/ml/linux-doc/20200409132437.11531-1-richard.weiyang@gmail.com/>) that the title of [this document](<https://elixir.bootlin.com/linux/v5.12.8/source/Documentation/RCU/RTFP.txt>) required a spelling correction; the [response](<https://lwn.net/ml/linux-doc/20200409134634.GL17661@paulmck-ThinkPad-P72/>) this time politely declined the patch, but encouraged the poster by saying ""rest assured that you did get [the] point!"". 

Anybody who has spent any amount of time trying to function in a non-native language understands that humor can be difficult. It is deeply tied to both the language it is expressed in and the specific context where it is found. Words that fluent speakers find funny can simply fly over the head of those who are less proficient, and attempts to be funny in a non-native language do not always go well. Your editor has a hard time getting a laugh from native English speakers; he has learned that it's better not to even try in other settings. 

Source code is meant to be precise and understandable. Comments are there to help when the code itself is not enough to explain why a particular action is taken. When those comments contain plays on words, creative misspellings, or jargon-laden inside jokes, they will become less clear to people who may be struggling to understand even plainly written English. Such readers will trip over the language in question, and some of them may then be moved to try to "fix" it. The result is a certain amount of confusion and lost time. 

One possible solution could be to eliminate this kind of play on words as an impediment to global free-software development. As Zhen Lei [said](<https://lwn.net/ml/linux-kernel/5571d34f-1965-8d8e-3033-6352dd0dcaee@huawei.com/>) in the first discussion linked above: ""Linux is an open-source code for the world, so it's better to use a more common word."" If we stuck with common words, we would see less confusion, fewer "fixes" for intentional humor, and possibly better participation overall. 

But, remember, free-software development is supposed to be fun as well as useful; that is part of why many of us got into it in the first place. Turning a free-software project into a humor-free zone would make it distinctly less fun, which could be seen as an inclusiveness problem in its own right. The cure could be far worse than the disease. 

Fortunately, nobody is proposing any such thing. Most of us are agreed that life can only be improved by something that brings a little smile in the middle of an extensive code-reading session. With any luck at all, our code will continue to acquire the occasional comment that makes it clear that we do not take ourselves too seriously. 

But that means we will also continue to receive the occasional "fix" from readers who were unable to appreciate a humorous turn of phrase and wish to straighten it out in some way. Those will be the best occasions of all to demonstrate that, indeed, we do not take ourselves too seriously. A curt response from a developer who clearly feels that their time is being wasted may turn away an aspiring contributor who we would much rather keep. A kind response, explaining the phrasing in question, will help that contributor appreciate the humor, improve their language skills, and get them to the point where, someday, they will embed some humor in useful code of their own. Others, silently watching the conversation, may also benefit. 

For better or worse, English is the language of the free-software development community. English is not the native language of a majority of our contributors, though, and some of those contributors will struggle with it at times. But, with some patience, we can have our code and laugh at it too.
