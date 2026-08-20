---
title: "A panel discussion on the kernel's code of conduct"
url: https://lwn.net/Articles/772565/
date: "November 20, 2018"
category: "Development model-Developer conduct"
author: "By Jonathan Corbet November 20, 2018 Linux
Plumbers Conference"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 20, 2018

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2018-Linux_Plumbers_Conference>)

There has been a great deal of discussion around the kernel project's recently adopted code of conduct (CoC), but little of that has happened in an open setting. That changed to an extent when a panel discussion was held during the Kernel Summit track at the 2018 [Linux Plumbers Conference](<https://linuxplumbersconf.org/>). Panelists Mishi Choudhary, Olof Johansson, Greg Kroah-Hartman, and Chris Mason took on a number of issues surrounding the CoC in a generally calm and informative session. 

Kroah-Hartman began by apologizing for the process by which the code was adopted. Linus Torvalds wanted something quickly, Kroah-Hartman said, so the process was rushed and a lot of political capital was burned to get the code into the kernel. He has since been trying to make up for things by talking to a lot of people; while he apologized for how things happened, he also insisted that it was necessary to take that path. The "code of conflict" that preceded the current code was also pushed into the kernel over a period of about three weeks; "we have been here before", he said. 

The current status is that we have a code of conduct and an interpretation document that describes how the code will be viewed in the kernel community. There is an email alias for the reporting of problems, and work is underway to set up a group to handle problems that is separate from the [![\[panel\]](https://static.lwn.net/images/conf/2018/lpc/CoC-panel-sm.jpg)](<https://lwn.net/Articles/772570/>) Linux Foundation Technical Advisory Board (TAB), which is currently charged with that task. He noted that putting the TAB in that role is nothing new, though; the TAB was also responsible for handling complaints under the old code of conflict. 

Mishi Choudhary has been appointed as a mediator for difficult code-of-conduct issues. She introduced herself by congratulating the kernel community as a whole, saying that everybody celebrates our way of creating software. The kernel's success will lead to the continued growth of the kernel community, meaning that we will have a lot of new people to welcome. She is there to help in any way she can, she said. 

Though originally from India, Choudhary is now based in New York, where she has been working with the open-source community for twelve years as a lawyer for groups like the Free Software Foundation, the Apache Software Foundation, and the OpenSSL project. In her current role at the Software Freedom Law Center, she has to deal with conduct-related issues, so she has the experience to help. The important thing, she said, is to be excellent and kind to each other; fairness and due process are not in conflict with kindness. 

Kroah-Hartman jumped in with one other thing he wanted to mention: a lot of people have ideas for how they might like to change the code of conduct. One sentence that didn't work with our community (making maintainers responsible for enforcing the code) has been removed but, in general, he suggested that it would be better to work with [the upstream project](<https://www.contributor-covenant.org/>) to propose changes. Kroah-Hartman noted that many people have complaints about the author of the code of conduct; he responded that he, personally, has a number of disagreements with Richard Stallman, but still uses Stallman's license. We don't have to agree with the author of the code, he said, and we can expect it to improve over time. 

#### Objectives

Frank Rowand said that any objections to the author are beside the point, it is the political objectives behind the document that many find objectionable. Kroah-Hartman answered that he doesn't agree with much of Stallman's intent either, but that is not a problem. We are using the code for the value it provides to us, and not to further somebody else's political objectives. Rowand continued by saying that the code has an intent that is antithetical to many in the community, and that needs to be acknowledged. In particular, its anti-meritocracy objective does not sit well with many. Rik van Riel responded that we need to rethink what we mean by "meritocracy"; true merit includes working well with others. 

Mel Gorman said that we should treat the code as a specification, not an implementation. The kernel community is creating its own implementation of that specification, and we often deviate from specifications when it makes sense to us; this case is no different. Treating merit as a black-and-white quantity is dangerous, he added; it can be exclusionary and restrict junior developers who have the capacity to learn. We should not get too hung up on meritocracy, and we should not be too worried about the code of conduct. We just need to focus on our own implementation. He finished by noting that, since he is Irish, he has a way of dealing with people that's incompatible with almost everybody else. He has to mediate his responses; others should be able to do the same. 

Shuah Khan said that "meritocracy" means that we will not accept a reduction in the quality of our code. But we can reject code with polite language; indeed, we have always done that, with just a few exceptions. Kroah-Hartman said the biggest change is that there will be no more exceptions. 

Mauro Carvalho Chehab said that the changes that have been made — the removal of maintainer responsibility, the interpretation document, and the appointment of a mediator — are all important. But he is still unsure about how the code will interact with the law in different countries. He mentioned the right to be forgotten as a particular problem in this regard. Kroah-Hartman replied that lawyers at the Linux Foundation and elsewhere have long since concluded that the work done in the kernel community is public and that we cannot be expected to edit it out, so the right to be forgotten does not apply. Mason added that he can sympathize with people who have legal concerns; the TAB can try to connect people with lawyers to get answers to specific questions when the need arises. 

Rowand said that the real goal is to create a better and more inclusive community. What else can we do to solve the real problems that we have? Kroah-Hartman said that the Linux Foundation would be announcing some initiatives soon. He also mentioned the work that is being done to create a maintainer's handbook as a step in that direction. Laura Abbott said that, while supporting new developers is important, experienced developers also need to treat each other well. Kroah-Hartman said that he "got to play Linus for a month" and that the experience was not much fun. Every maintainer does things differently. There is a lot that we all can do to make life easier for other maintainers. 

Johansson said that, as a community, we have long optimized for keeping the people we care about around; we want them to be there in the future to support their code. Some of that pressure is being relieved by the new set of development tools and testing programs, though; it is easier to be sure that code is in good shape when it goes into the kernel and should help us to broaden the contributor base. 

#### Maintainer responsibility

Gorman warned that the community should be wary of placing responsibility for upholding the code of conduct on any specific individuals. The load on maintainers is high now; it will become worse if maintainers are made responsible for more tasks. There will always be members of our community, he said, who are not suited to mentoring contributors. When I see an example of somebody being "crucified" in a review, he said, we all have the responsibility to intervene. It is possible to tear a patch apart without ever explaining what would make the patch acceptable; such a review can be "very friendly" but also quite unhelpful. Complaining to the TAB will not help in such situations; instead, the best thing to do is to enter the conversation and help the contributor figure out what they need to do. It is not fair to put that responsibility on any one maintainer, or on the TAB. 

Dan Williams said that part of the problem is that he doesn't always have the time or ability to mentor contributors, or he doesn't know what to suggest to them without trying to solve the problem himself. It's not always easy to see how you want the problem solved in the end. Johansson said that, if you're the only one who can review patches in your subsystem, you have a growth opportunity for somebody else. Try to find that person and bring them up to speed; that is how we develop new maintainers. Ted Ts'o added that the only power in a volunteer organization is the ability to say "no"; it's nearly impossible to compel people to do things. So telling maintainers to make something happen will not work. The only way is for people to improve things themselves and to recruit others to help. 

Mason said that almost all patches receive one of four possible answers: "yes", "no", "no with reasons", and "no with reasons and a long story about how terrible the patch is". He would like to get rid of the last type of answer, which is not helpful. If the code of conduct can do that, he said, we will be better off in the end. 

Abbott said that most code-of-conduct issues are things that maintainers and developers can handle themselves. Escalation is not needed, good judgment is enough. The code is mostly a way to establish what we value and a plan in case we have to deal with a more serious issue someday; she hopes that we will never need it. Choudhary also hoped that she would never be needed. Each community figures out how it wants to resolve these problems, she said; the kernel community will be able to do the same. Johansson said that the TAB isn't just there as a sort of police force; anybody is encouraged to reach out with questions or for help. 

Jes Sorensen said that, while there has been a lot of talk about how we need to be helpful, one of our bigger problems is contributors who refuse to listen. Some of them are just stubborn; others are overtly trying to provoke a reaction. How do we cope with such people? Kroah-Hartman said that this problem is outside of the code of conduct. We have dealt with such problems in the past, with the worst offenders being banned from the mailing lists. Sorensen was unconvinced, though, citing problems with contributors who have worked on a body of code for years and keep pushing it even after it has been rejected. Such problems will happen again, he said. 

A solution to that problem will have to wait, though, as the session ran out of time and came to an end. The overall feeling was one of wary acceptance; kernel developers generally agree with the goals of the code of conduct (as expressed within the community) and hope that the actual outcome will consistent with those goals. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to LPC.]
