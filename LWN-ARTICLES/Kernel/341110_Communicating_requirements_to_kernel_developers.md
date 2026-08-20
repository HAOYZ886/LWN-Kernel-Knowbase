---
title: Communicating requirements to kernel developers
url: https://lwn.net/Articles/341110/
date: "July 14, 2009"
category: Development model
author: "By Jonathan Corbet July 14, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 14, 2009

The 2009 kernel summit is planned for October in Tokyo. Over the years, your editor has observed that the discussion on what to discuss at the summit can sometimes be as interesting as the summit itself. Recently, the question of how user-space programmers can communicate requirements to the kernel community was raised. The ensuing discussion was short on definitive answers, but it did begin to clarify a problem in an interesting way. 

For the curious, the entire thread can be found in [the ksummit-2009-discuss archives](<https://lists.linux-foundation.org/pipermail/ksummit-2009-discuss/2009-July/thread.html>). Matthew Garrett [started things](<https://lwn.net/Articles/341117/>) this way: 

I've just run a session at the desktop summit in Gran Canaria on the functionality that userspace developers would like to see from the kernel. Some interesting things came out of it, but one of the major points was that people seemed generally unclear on how they could communicate those requirements to kernel developers. Worth discussing? 

Dave Jones's [response](<https://lwn.net/Articles/341118/>) was instructive: 

What exactly is the problem ? They know where linux-kernel@vger.kernel.org is, so why aren't they talking to us there? 

To developers who are used to the ways of linux-kernel, and who are well established in that community, a question like this might make sense. If one were to poll developers who do not normally hang out in the kernel community, though, one might get an answer something like this: 

  1. The volume on linux-kernel is far too high for ordinary people to cope with. 

  2. Even if we could keep up with linux-kernel, the volume is still likely to bury anything we might post there. 

  3. Kernel people speak their own language, making it hard to follow discussions, much less participate in them. 

  4. If somebody does notice our request, they will probably flame it to a cinder without necessarily taking the time to understand it first. 

  5. If they don't flame it, they will probably tell us to send a patch, but we're not kernel developers and thus not in a position to do that. 

There can be no doubt that some communications problems can be easily blamed on the requesting side. If a feature request is phrased as a demand, it is unlikely to be received well. Kernel developers are beholden to demands from their employers, but from nobody else; like most other developers, they take a dim view of people who feel entitled to free work just because they want it. A classic example here would be the early Carrier Grade Linux specifications produced by OSDL; they read like a "to do" list handed to the kernel community, even if OSDL eventually claimed that it was not intended that way. 

Another problem can be poorly-expressed requirements. Consider [the early TALPA proposal](<http://lwn.net/Articles/292778/>), which posted a very clear set of low-level requirements. Unfortunately, they were _too_ low-level, requiring features like the ability to intercept file close operations. Instead, TALPA (now [fanotify](<http://lwn.net/Articles/339399/>)) needed to express requirements like "we need a clean way to support proprietary malware-scanning software, and this is why." That disconnect set the project back significantly, and could well have killed a project whose developers showed less persistence or less willingness to learn. Clearly expressing requirements at the right level is never an easy task, but it's crucial. 

Finally, some ideas just don't make sense in the kernel. Perhaps they cannot be implemented in a way which avoids security problems, does not break other features, and does not create long-term maintenance problems. Or perhaps there are better solutions in user space. A developer who goes to linux-kernel with this kind of request is likely to go away feeling like kernel developers are completely unwilling to listen to reason. 

All of the above notwithstanding, there is some recognition that user-space developers have real difficulties in bringing requirements to the kernel community. Kernel developers tend to be busy, focused on their own projects, and not always entirely open to requests from outside the community. There is no mechanism for tracking feature requests, so it is very easy for them to be buried in the flood of email. The tone of discussions can be harsh, even though it truly has improved over the years. And so on. 

This is not good. The kernel exists to provide for the needs of user space; if the kernel development community is not hearing what those needs are, it can only fail to satisfy them. So thinking about how to make it easier for user-space developers to communicate their requirements would seem to be worthwhile; chances are that space will be made at the summit for that topic. But there is no need to wait for the summit to start talking about how things could be improved. 

Matthew Wilcox [suggested](<https://lwn.net/Articles/341123/>) the creation of a document on how user-space developers can interact with the kernel community. The idea makes sense (your editor may just try to help there), but this is not a problem which can be solved by documents alone. 

James Bottomley [described](<https://lwn.net/Articles/341124/>) three broad categories of users needing changes to the kernel: 

  1. Sophisticated developers who can write their own kernel extensions. 
  2. Users who can get a kernel developer interested in their desired additions. 
  3. Users who want features that no developers are interested in. 

James points out that categories 1 and 2 can be helped with documentation and general outreach. He worries, though, that we have no way to help the third category of users, who are generally left with no way to get the kernel changed to meet their needs. 

Ted Ts'o had [a different taxonomy](<https://lwn.net/Articles/341125/>) which he has put forward as a way to help understand the problem: 

  1. Core kernel developers (or those who have access to such people). Core developers have the advantage of a high degree of trust in the community; that allows them to get features into the kernel with a relatively small amount of trouble. They are able to merge code which might well not pass muster if it came from a different source. 

  2. Competent, but non-core kernel developers (and, again, people who have access to them). These developers have to work harder to justify their changes, but they are generally in a position to get changes merged as long as the work is good. 

  3. Potentially competent developers with "patently bad design taste." Ted suggests that the frank nature of the kernel review process is intended mainly to weed out bad patches from this source. 

  4. Users with no access to kernel development expertise, who must thus try to convince somebody else in the community to implement their desired feature for them. Ted divided this category into two subcategories, depending on whether there is an active kernel developer working in the user's area of interest or not. 

Ted's thought is that this taxonomy can help users to understand why certain patches and ideas are treated the way they are. It can also be used to help develop ways to reach out to each specific group of users. Certainly the different groups need to hear different messages. One could argue that the existing documentation should be sufficient for people with kernel development skills, but there is relatively little help out there for those who must find a developer to do their work for them. 

It is that last group which is most likely to be intimidated by the prospect of walking into linux-kernel and asking for features. The kernel community could really use a person who would take on the task of working with these users, helping them to clarify their requirements, connecting them with the appropriate developers, and tracking requests. The good news is that we do have such a person; the bad news is that it's Andrew Morton, who has one or two other things to do as well. The community would benefit from a person who worked something close to full time on the task of helping users and user-space developers get what they need from the kernel. That sort of position tends to be very hard to fund, though; as a result, it tends to stay vacant. 

As was noted at the outset, this conversation did not produce much in the way of concrete conclusions. It is far from complete, though. If not before, it will be resumed in October at the full summit. Needless to say, your editor plans to be there; stay tuned.
