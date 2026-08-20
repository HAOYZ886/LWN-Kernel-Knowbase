---
title: "ELC: Morton and Saxena on working with the kernel community"
url: https://lwn.net/Articles/278647/
date: "April 21, 2008"
category: "Development model-Community; Embedded systems"
author: "By Jake Edge April 21, 2008"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
April 21, 2008

In many ways, Andrew Morton's keynote set the tone for this year's [Embedded Linux Conference](<http://embeddedlinuxconference.com/elc2008/index.html>) (ELC) by describing the ways that embedded companies and developers can work with the kernel community in a way that will be "mutually beneficial". Morton provided reasons, from a purely economic standpoint, why it makes sense for companies to get their code into the mainline kernel. He also provided concrete suggestions on how to make that happen. The theme of the conference seemed to be "working with the community" and Morton's speech provided an excellent example of how and why to do just that. 

Conference organizer Tim Bird introduced the keynote as "the main event" for ELC, noting that he often thought of Morton as ""kind of like the adult in the room"" on linux-kernel. Readers of that mailing list tend to get the impression that there's more than one of him around because of all that he does. He also noted that it was surprising to some that Morton has an embedded Linux background--from his work at Digeo. 

Morton believes that embedded development is underrepresented in kernel.org work relative to its economic importance. This is caused by a number of factors, not least the financial constraints under which much embedded development is done. An exceptional case is the chip and board manufacturers who have a big interest in seeing Linux run well on their hardware so that they can attract more customers. But even those do not contribute as much as he would like to see to kernel development. 

An effect of this underrepresentation is a risk that it will tilt kernel development more toward the server and desktop. The kernel team is already accused of being server-centric, and there is some truth to that, ""but not as much as one might think"". Kernel hackers do care about the desktop as well as embedded devices, but without an advocate for embedded concerns, sometimes things get missed. 

Something Morton would like to see is a single full-time "embedded maintainer". That person would serve as the advocate for embedded concerns, ensuring that they didn't get overlooked in the process. An embedded maintainer could make a significant impact for embedded development. 

Not all kernel contributions need to be code, he said. There is a need just to hear the problems that are being faced by the embedded community along with lists of things that are missing. "Senior, sophisticated people" are needed to help prioritize the features that are being considered as well. Morton often finds out things he didn't know at conferences, things that he should have known about much earlier: ""That's bad!"" 

Morton is trying to incite the embedded community to interact with the kernel hackers more on linux-kernel. He said that a great way to get the attention of the team is to come onto the mailing list and make them look bad. Unfavorable comparisons to other systems or earlier kernels, for example, especially when backed up with numbers, are noticed quickly. He said that it is important to remember that the person who makes the most noise gets the most attention. 

One of the areas that he is most concerned about is the practice of "patch hoarding"--holding on to kernel changes as patches without submitting them upstream to the kernel hackers. It is hopefully only due to a lack of resources, but he has heard that some are doing it to try and gain a competitive advantage. This is simply wrong, he said, companies have a ""moral if not legal obligation"" to submit those patches. 

[PULL QUOTE:  The code will be better because of the review done by the kernel hackers; once it is done, the maintenance cost falls to near zero as well. He also touted the competitive advantage, noting that getting your code merged means that you have won--competing proposals won't get in.  END QUOTE]

There are many good reasons for getting code merged upstream that Morton outlined. The code will be better because of the review done by the kernel hackers; once it is done, the maintenance cost falls to near zero as well. He also touted the competitive advantage, noting that getting your code merged means that you have won--competing proposals won't get in. Being the first to merge a feature can make it easier on yourself and harder on your competition. 

There are downsides to getting your code upstream as well. Most of those stem from not getting code out there early enough for review. The kernel developers can ask for significant changes to the code especially in the area of user space interfaces. If a company already has lots of code using the new feature and/or interface, it could be very disruptive; ""sorry, there's no real fix for that except getting your code out early enough"". 

Another downside that companies may run into is with competitors being brought into the process. Morton and other kernel hackers will try to find others who might have a stake in a new feature to get them involved so that everybody's needs are taken into account. This can blunt the "win" of getting your feature merged. Some are also concerned that competitors will get access to the code once it has been submitted; "tough luck" Morton said, everything in the kernel is GPL. 

Morton had specific suggestions for choosing a kernel version to use for an embedded project. 2.6.24 is not a lot better than 2.4.18 for embedded use, but it has one important feature: the kernel team will be interested in bugs in the current kernel. He suggests starting with the current kernel, upgrading it while development proceeds, freezing it only when it is time to ship the product. 

He also suggests that a company create an internal kernel team with one or two people who are the interface to linux-kernel. This will help with name recognition on the mailing list, which will in turn get patches submitted more attention. Over time, by participating and reviewing others' code, the interface people will build up "brownie points" that will allow them to call in favors to get their code reviewed, or to help smooth the path for inclusion. 

The kernel.org developers appear to give free support, generally very good support, Morton said, but it is not truly free. Kernel hackers do it as a "mutually beneficial transaction"; they don't do it to make more money for your company, they do it to make the kernel better. Morton is definitely a big part of that, inviting people to email him, especially if ""five minutes of my time can save months of yours"". 

The decision about when to merge a new feature is hard for some to understand. Many consider Linux a dictatorship, which is incorrect, it is instead ""a democracy that doesn't vote"". The merge decision is made on the model of the "rule of law" with kernel hackers playing the role of judges. Unfortunately, there are few written rules. 

Some of the factors that go into his decision about a particular feature are its maintainability, whether there will be an ongoing maintenance team, as well as the general usefulness of the feature. Depending on the size of the feature, an ongoing maintenance team can be the deciding factor. It is not so important for a driver, but a new architecture, for example, needs ongoing maintenance that can only be done by people with knowledge of and access to the hardware. 

MontaVista kernel hacker, Deepak Saxena, gave a presentation entitled "Appropriate Community Practices: Social and Technical Advice" later in the conference that mirrored many of Morton's points. He showed some examples of hardware vendors making bad decisions that got shot down by the kernel developers, mostly because they didn't "release early and release often". There is a dangerous attitude that ""it's Linux, it's open source, I can do anything I want"" which is true, but won't get you far with the community. 

Saxena has high regard for the benefits of working with the system: if your competitor is active in the community, they are getting an advantage that you aren't. Like Morton, he believes that some members of the development team need to get involved in kernel.org activities. ""The community is an extension of your team, your team is an extension of the community"". 

He also has specific advice for hardware vendors: avoid abstraction layers, recognize that your hardware is not unique, and think beyond the reference board implementation. Generalizing your code so that others can use it will make it much more acceptable, as will talking with the developers responsible for the subsystems you are touching. Abstraction layers may be helpful for hardware vendors trying to support multiple operating systems, but they make it difficult for the kernel hackers to understand and maintain the code. The kernel.org folks are not interested in finding and fixing bugs in an abstraction layer. 

He also points out additional benefits of getting code merged. Once it is in the kernel, the company's team will no longer have to keep up with kernel releases, updating their patches to follow the latest changes. The code will still need to be maintained, but day-to-day changes will be handled by the kernel.org folks. An additional benefit is that the code will be enhanced by various efforts to automatically find bugs in mainline kernel code with tools like [lockdep](<http://lwn.net/Articles/185666/>). 

It is clear that the kernel hackers are making a big effort to not only get code from the embedded folks, but also some of their expertise. There are various outreach efforts to try and get more people involved in the Linux development process; these two talks are certainly a part of that. By making it clear that there are benefits to both parties, they hope to make an argument that will reach up from engineering to management resulting in a better kernel for all.
