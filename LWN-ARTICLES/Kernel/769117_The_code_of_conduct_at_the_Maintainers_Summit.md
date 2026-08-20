---
title: The code of conduct at the Maintainers Summit
url: https://lwn.net/Articles/769117/
date: "October 23, 2018"
category: "Development model-Developer conduct"
author: "By Jonathan Corbet October 23, 2018 Maintainers Summit"
---

By **Jonathan Corbet**  
October 23, 2018

* * *

[Maintainers Summit](<https://lwn.net/Articles/769260/>)

The 2018 Kernel Maintainers Summit convened in Edinburgh, UK on October 22 with a number of things to discuss, but the top subject on most minds was [the recently (and hastily) adopted code of conduct](<https://lwn.net/Articles/765108/>). Linus Torvalds made his reentry into the kernel community with a discussion of how we got to the current state of affairs, and the assembled maintainers had a relatively good-natured discussion on how this situation came about and where things can be expected to go from here. 

Torvalds started by noting that the conduct issue is not a new one; it has been "festering in the community" for years. The immediate cause of his decision to take a break and bring in the code of conduct was knowledge that [The New Yorker article](<https://www.newyorker.com/science/elements/after-years-of-abusive-e-mails-the-creator-of-linux-steps-aside>) was coming; he noted that, contrary to what was written there, the author never tried to contact him. That article led to a number of discussions with friends and others; Torvalds concluded that the best way to "head things off" was to announce some changes with [the 4.19-rc4 release](<https://lwn.net/Articles/764901/>). He acknowledged that this was done in private and it was rushed; it did not follow the usual open-source model. After the fact, he admitted to not being sure that the article justified all of the heartache that preceded it. But, as James Bottomley noted, the -rc4 announcement and adoption of the code of conduct did cause the article to be rewritten. 

The task of writing that announcement was not fun, Torvalds added, but contrary to some speculation on the net, he did write it all himself. He suggested that anybody who needs to write a message of that nature take a few days to think about it. 

Whether or not the article justified the trouble, he became convinced that he had taken the right course after about a week of reading the "vile garbage" that came from people who were opposed to it. He even saved a couple of particularly special emails that were sent to him; they dispelled any doubts that he was on the right side. From here, he only had a couple of suggestions. While he agrees with [the addition of the interpretation document](<https://lwn.net/Articles/769056/>) and the changes to the code itself, now would be a good time to stop making changes and just let things be. There are a lot of people worried about hypothetical situations, but we shouldn't make more changes unless and until something happens. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Steve Rostedt interjected that the code of conduct is not "our code" and that it would be better to move to something that better reflects our community. Torvalds concurred that a lot of people do not necessarily agree with the author of the [Contributor Covenant](<https://www.contributor-covenant.org/>), upon which the kernel's code is based, but that agreement is not necessary; the code itself is good, he said, and we should resist the temptation to mess with it further. There should be no more hidden emails about it; nobody is entirely happy with it, but we can live with it. Greg Kroah-Hartman added that many other projects have used it; adopting it is like picking a well-known license rather than writing a new one. 

Torvalds went on to say that, as far as he is concerned, if the code of conduct ever triggers, it will indicate a big problem; he does not want the code to be an ongoing issue in the community. To that end, he asked the assembled group to watch his emails and let him know if things start to get close to the edge. He has, in fact, installed a profanity filter on his outgoing mail, but it is easy to be impolite without cursing. Kroah-Hartman noted that the previous "code of conflict" had been around for several years; it only generated three reports ever, none of which had any real substance. The community has a good history of doing sane things in this area; we also have a professional mediator, funded by the Linux Foundation, to help in that regard. Contact information for the mediator can be found in the interpretation document. 

Kees Cook said that conversations between kernel developers can be scary, especially to relatively new members of the community who see them from the outside. Having the code of conduct tells these people that there is somebody they can talk to if the need arises. Bottomley added that adopting the code will help to convince the outside world that the community has gotten better. Cook noted that, "two rants ago", Torvalds apologized to him afterward, and the last one was a joke, so things were already getting better. 

Ted Ts'o said that many of these interactions are context-dependent. Ten years ago, an effort was made to encourage Japanese developers in particular; part of that was sensitizing developers in the community that, in some cultures, direct criticism can lead to strong feelings of shame. Once people know about issues like that, he said, they tend to be more careful. On the other hand, Christoph Hellwig said that the "I love your patch" message from the 0day robot (which reports problems with patches) was offensive to many in a different way. 

The documents associated with the code of conduct, Ts'o said, should not be seen as an absolute declaration of how things will be; instead, they are a symbol describing what we are aiming for. Grant Likely added that the community has not really changed much, but now we have thought about what to do when things go bad and have a way to deal with such situations. Laura Abbott added that having the code will help new developers feel that the welcome they get at the beginning will continue to be there as they grow into the community. 

Peter Zijlstra admonished, though, that if a developer continually ignores his feedback, he'll eventually stop being nice. Others responded that it is OK to say that the code is wrong, but one can't call the developer an idiot. But what is to be done when the person, who is ignoring feedback, is the real problem? It is still not acceptable to attack the person, Dirk Hohndel said. Instead, in the worst cases, the only real alternative may be to simply ignore the patches. 

Ted Ts'o wondered about difficult cases like [the current effort to get the WireGuard patches merged](<https://lwn.net/Articles/761939/>). There is no real misbehavior happening there, it is just friction between developers. One of the reasons he started the Kernel Summit in the first place was to get kernel developers to meet each other; it's much harder to get mad at somebody you've shared a beer with. Arranging such meetings is harder now, since the community is so much bigger. In response, we all have to work harder to assume good faith on the part of other developers. 

There was a fair amount of discussion on how it might be possible to get more new developers to conferences. The invitation that developers would receive to the older, larger Kernel Summit turn out to be important at a lot of companies, so it may be worthwhile finding a way to revive them. There is a lot of thought that needs to go into conferences in general, though; there are far too many conferences, so it has become difficult for aspiring developers to meet with established members of the community. 

Returning to the code of conduct, Ts'o noted that the mandates on maintainers raised a lot of hackles. The purpose wasn't really to create more work for maintainers, though, or to turn them into police officers; instead, it was an expression of the idea that maintainers need to lead by example. We should all do that, he said, and to try to talk to people when we see borderline emails on the lists. 

Mauro Carvalho Chehab expressed some worries that parts of the code of conduct could be seen as a binding contract in Brazil. Evidently, though, lawyers at the Linux Foundation have reviewed it with that concern in mind and concluded that it is not the case. There were some questions about what the kernel community would do if the upstream Contributor Covenant code were to change; the answer is that the changes will be evaluated when they happen. 

As things wound down I tried to reemphasize the point that the time for private conversations around the code of conduct passed a while back. It was agreed that any future discussions would happen in a public forum. Kroah-Hartman added that, in a couple of years, it will likely become necessary to add some other sort of sensitive, policy-related document using a similar process, though. 

When, it was asked, is Torvalds returning to the community? He answered that he is already back; he has gotten some pull requests and intends to return to working normally. It was nice, he said, to have Kroah-Hartman take over for a while and give him a break, though. Kroah-Hartman has write permission to the repository now, so he may be asked to take over again at some point. Torvalds noted dryly, referring to the mixup that got the Maintainers Summit moved to Edinburgh in the first place, that he has a vacation coming up soon where that might be welcome. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to the event.]
