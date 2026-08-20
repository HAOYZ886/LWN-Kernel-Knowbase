---
title: An open letter to Evgeniy Polyakov
url: https://lwn.net/Articles/308566/
date: "November 25, 2008"
category: Development model
author: "By Jonathan Corbet November 25, 2008"
---

By **Jonathan Corbet**  
November 25, 2008

[_Editor's note: the following article may look like a message to a specific kernel developer, but it is really about the development process in general. Over the years, your editor has seen too many worthy hackers run into development process problems; the end result is often that we lose that person's contributions. We are not so rich that we can afford that sort of loss. The desire to prevent such problems was the motivation behind your editor's recently-written[development process document](<http://ldn.linuxfoundation.org/book/how-participate-linux-community>) \- and this letter._] 

Dear Evgeniy, 

Your editor has chosen to write to you in a public manner because he hates to see talented developers get frustrated with the kernel process and storm off. We do not have an excess of capable hackers, especially those who can work at your level. Losing one hurts. Your editor hopes that this eventuality can be avoided in this case - for you, and for others who may be encountering the same sort of frustrations you are. Getting code into the kernel can be a pain, sometimes. That said, some 1160 developers have managed it since the opening of the 2.6.28 merge window in October. It is possible to get code merged with sufficient care. 

You first posted your distributed storage (DST) patch back in 2007; LWN [took a look at it](<http://lwn.net/Articles/246272/>) at that time. Since then, this code has come a long way. Beyond the basic task of exporting (and accessing) storage volumes across the net, this code claims "bullet-proof memory allocations," zero-copy transport, failover recovery with full transaction support, support for IPv6 and beyond, and a number of features including encrypted data channels. And, it is said, this code is fast. In general, it looks like good stuff. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

You have posted the DST code on the mailing lists a number of times - too many, apparently, for your tastes. Frustration with the process appears to have led to the behavior described in [your recent weblog post](<http://www.ioremap.net/node/72>): 

To understand the roots of this issue, I made a simple experiment with the previous DST release. I added following lines into the patch to catch reviewer's eyes: 

```
ass licker
        static char dst_name[] = "Successful erackliss screwing into";
```

As you may expect, this does not compile and thus was never read by the people who are subscribed to the appropriate mail lists. I got one private mail about this fact for the whole week. The same DST code (without above lines) was sent public first time more than month ago and was resent 3 times after that. 

That's why I do not care about DST inclusion anymore. I do not care about its linux-kernel@ feedback. 

So, because the fourth posting of identical code in one month received little attention, DST now risks joining Kevents, network channels, network tree memory management, asynchronous crypto, and more in that place where dusty, out-of-tree stuff lives. This would not be a good outcome. So let us look at what can be done to avoid that - for your sake, for DST users' sake, and for the sake of other developers who may follow. 

One way to get more reviews for your code is to pay attention to what those reviewers are saying. Andrew Morton [spent some time](<https://lwn.net/Articles/308570/>) on DST back in October. He had a number of concrete requests - such as documenting the user-space ABI and the network protocol - which have not been satisfied. He also asked for better code documentation in general: 

So please. Go through all the code and make it tell a story. Ask yourself "how would I explain all this to a kernel developer who is sitting next to me". It's important, and it's an important skill. 

The [November 25, 2008 version](<http://lwn.net/Articles/308538/>) of DST still does not tell that story, and that makes it very hard for other developers to understand. Code review, as you know, is in critically short supply in most free software projects. Getting reviews for difficult-to-understand code is hard, especially when it is a large body of complex code which occupies a niche in which relatively few developers have expertise. So it's not surprising that your most recent comment involved white space - anybody can make that kind of review without any need to actually understand what's going on. 

Not only does your patch not tell a story, but the individual pieces of it do not even contain changelogs. For a patch set marked "consider for inclusion," that is a fatal error. Playing along with the system on things like that can seem like a waste of time, especially if you hold out no real hope of the patch being merged, but it is a necessary sign of respect for the people you are asking to consider the patch. No maintainer will accept a patch without a changelog. 

While we're on the topic of documentation, your kernel configuration help text reads, in its entirety: 

This driver allows to create a distributed storage block device. 

You owe your users a little bit more than that. Why might they want to use DST? Where can they get the associated tools? This, too, is a fatal error for any substantive kernel change. 

And, while we're still somewhat on the subject of reviews: Andrew naturally called out the generic-looking thread pool implementation buried deep within DST; shouldn't it pulled out and made more generic? Your [response](<https://lwn.net/Articles/308575/>) can be paraphrased as "I can't be bothered to get the API past the review process, which, in any case, is biased toward those who are 'closer to the high end'." But pulling out this code and merging it separately might be the ideal starting point for getting the larger patch set into the kernel. A generic thread pool hiding within a storage device driver, instead, will be an ongoing impediment to inclusion. 

Then there is the issue of motivation: why should the kernel developers want to merge this patch? Who are the users of it - do you have users now? How does it compare to other distributed storage technologies already in the kernel? What's the performance like - can you post some benchmark results? As it stands, DST looks like a nice piece of technology, but its benefits are still unclear. Tell that story, and the level of interest may well go up. 

Finally, your editor would like to counsel patience. Some patches just take longer than others to find their way in the kernel. That is especially true of complex patches which touch on issues like memory management and which add new user-space ABIs. As a close-to-home example, look at David Howells's FS-Cache code, recently [reposted](<http://lwn.net/Articles/308113/>) for consideration. The [first LWN article](<http://lwn.net/Articles/100321/>) on this code was published more than four years ago. David is probably getting a little tired of maintaining this code out-of-tree, but he sticks with it, responds to reviews, and appears to be getting closer to inclusion. 

Evgeniy, you appear to be a brilliant and productive hacker. You charge into places that scare off most kernel developers, and you always come back out with something interesting. We _need_ developers like you. But we need developers like you who can work with the process - no matter how frustrating it gets. The kernel process is certainly far from perfect, but it is built around a set of principles which have served us well for many years. You could easily rise up through that process to become one of the "high end" developers who, you say, have an easier time getting code merged. Or you could take your marbles and storm home, making snide comments about reviewers on the way. But that would not be good for anybody involved. 

(See also: [Evgeniy's response](<http://www.ioremap.net/node/74>) to this article.)
