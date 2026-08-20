---
title: "Pointy-haired kernel hackers?"
url: https://lwn.net/Articles/240758/
date: "July 11, 2007"
category: Development model
author: 
---

Don Marti attended Greg Kroah-Hartman's Linux Symposium talk on the kernel development process; he wrote an informative article (titled [Linux contributor base broadens](<http://www.linuxworld.com/news/2007/062807-linux-contributor-base.html>)) about it. The article states: 

In the latest kernel release, the most active 30 developers authored only 30% of the changes, while two years ago, the top 20 developers did 80% of the changes, he said. Kroah-Hartman himself is now doing more code reviewing than coding. "That's all I do, is read patches these days," he said. 

An important part of this is that Greg presented this change as a good thing. The kernel has a far broader developer base than it once did, with patches for any given release coming from almost 1000 different people. We have a growing development community which is healthy and robust. 

Seeing what the mainstream media makes of things can be great fun sometimes. This time around, ComputerWorld UK picked up Don's article, running the same text but giving it a new title: [Are top Linux developers losing the will to code?](<http://www.computerworlduk.com/technology/hardware/processors/news-analysis/index.cfm?articleid=604&pn=1>) Slashdot picked it up under that title, then Wired chimed in with [Core Linux Developers Stuck In Middle Management Mode](<http://blog.wired.com/monkeybites/2007/07/core-linux-deve.html>), complete with a picture of a necktie-wearing employee wielding a stapler and a telephone. The prize must go to the Jem Report's [The coders and the talkers](<http://www.thejemreport.com/mambo/content/view/320>), though; this article breaks new ground completely: 

Linus' [sic] job is leaning more towards spokesman than programmer. He's been a relatively effective manager up until now, but I think that effectiveness will begin to erode rapidly with time. The further you get away from the actual work, the less you are able to accurately judge the appropriateness of other people's work. You need to stay in the game -- you need to keep your skills in condition. If you don't, you might understand the theory pretty well, but you'll get further and further away from being in touch with its application. Linus has become more of a talker and less of a coder. 

It seems we have trouble here. While we weren't looking, the Linux kernel drifted into a Dilbertesque realm and is now controlled by people who don't actually create software anymore. World Domination, it would seem, is now in grave doubt. 

Or perhaps all of this is just silly nonsense, an extreme extrapolation taken from a couple of sentences spoken at a Linux developer conference. 

If one wanted to investigate this subject further, a good starting place might be the [2.6.22 changelog](<http://lwn.net/Articles/240666/>); there one can see just how many patches our pointy-haired top-level maintainers have contributed over the latest development cycle. Here's a subset: 

> Developer| Patches  
> ---|---  
> Andi Kleen | 70  
> Andrew Morton | 79  
> David Miller | 193  
> Greg Kroah-Hartman | 14  
> James Bottomley | 13  
> Linus Torvalds | 20  
> Russell King | 61  
  
One could add more names to the list, but the end result would be about the same: the top-level kernel maintainers are not among the most prolific contributors (except maybe for David Miller - but, then, he's an exception in many regards), but neither are they absent from the game. They are still hacking on the code and cranking out the patches. 

From some of the articles that have been posted, one might think that subsystem maintainers spend the rest of their time attending meetings and writing weekly reports. What they are actually doing is working with patches - lots of patches - and with developers. A subsystem maintainer must review code, decide whether it is appropriate for the kernel, and, if not, give the developer guidance on how to make it better. Incoming patch streams must be merged together, and decisions must be made on which ones are ready for any given development cycle. It is a task which requires the maintainers to keep their hands deep in the code. Subsystem maintainers who do not touch the code, and who do not maintain a deep understanding of how the kernel works are not likely to remain maintainers for very long. 

So the broadening of the kernel development community - and the associated need for more work by subsystem maintainers - is not really costing us our best developers. They are not "losing the will to code." The things that cost us developers are elsewhere: a somewhat adversarial process which turns off some people, general burnout, or getting a job which takes their attention away from contributing to the community. All very normal stuff. In the Linux kernel community we may have our share of [Dogberts](<http://en.wikipedia.org/wiki/Dogbert>), but we need not lose much sleep over the threat of pointy-haired subsystem maintainers bringing the process to a halt. Instead, they have helped the kernel development process to scale to a level beyond that of almost any other software development project anywhere; that is a good thing, not a sign of trouble.
