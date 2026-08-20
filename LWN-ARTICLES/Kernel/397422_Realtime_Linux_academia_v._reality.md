---
title: "Realtime Linux: academia v. reality"
url: https://lwn.net/Articles/397422/
date: "July 26, 2010"
category: Academic systems; Realtime
author: "July 26, 2010 This article was contributed by Thomas Gleixner"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 26, 2010

This article was contributed by Thomas Gleixner

The [20th Euromicro Conference on Real-Time Systems (ECRTS2010)](<http://ecrts.eit.uni-kl.de/ecrts10>) was held in Brussels, Belgium from July 6-9, along with a series of satellite workshops which took place on July 6\. One of those satellite workshops was [OSPERT 2010](<http://www.artist-embedded.org/artist/Overview,1909.html>) \- the Sixth International Workshop on Operating Systems Platforms for Embedded Real-Time Applications, which was co-chaired by kernel developer Peter Zijlstra and Stefan M. Petters from the Polytechnic Institute of Porto, Portugal. Peter and Stefan invited researchers and practitioners from both industry and the Linux kernel developer community. I participated for the second year and tried, with Peter, to nurse the discussion between the academic and real worlds which started last year at OSPERT in Dublin. 

Much to my surprise, I was also invited to give the opening keynote at the main conference, which I titled "The realtime preemption patch: pragmatic ignorance or a chance to collaborate?". Much to the surprise of the audience I did my talk without slides, as I couldn't come up with useful ones as much as I twisted my brain around it. The organizers of ECRTS asked me whether they could publish my writeup, but all I had to offer were my scribbled notes which outlined what I wanted to talk about. So I agreed to do a transcript from my notes and memory, without any guarantee that it's a verbatim transcript. Peter at least confirmed that it matches roughly the real talk. 

#### An introduction

First of all I want to thank Jim Anderson for the invitation to give this keynote at ECRTS and his adventurous offer to let me talk about whatever I want. Such offers can be dangerous, but I'll try my best not to disappoint him too much. 

The Linux Kernel community has a proven track record of being in disagreement with - and disconnected from - the academic operating system research community from the very beginning. The famous Torvalds/Tannenbaum debate about the obsolescence of monolithic kernels is just the starting point of a long series of debates about various aspects of Linux kernel design choices. 

One of the most controversial topics is the question how to add realtime extensions to the Linux kernel. In the late 1990's, various research realtime extensions emerged from universities. These include KURT (Kansas University), RTAI (University of Milano), RTLinux (NMT, Socorro, New Mexico), Linux/RK (Carnegie Mellon University), QLinux (University of Massachusetts), and DROPS (University of Dresden - based on L4), just to name a few. There have been more, but many of them have only left hard-to-track traces in the net. 

The various projects can be divided into two categories: 

  1. Running Linux on top of a micro/nano kernel 
  2. Improving the realtime behavior of the kernel itself 

I participated in and watched several discussions about these approaches over the years; the discussion which is burned into my memory forever happened in summer 2004. In the course of an heated debate one of the participants stated: ""It's impossible to turn a General Purpose Operating System into a Real-Time Operating System. Period."" I was smiling then as I had already proven, together with Doug Niehaus from Kansas University, that it can be done even if it violates all - or at least most - of the rules of the academic OS research universe. 

But those discussions were not restricted to the academic world. The Linux kernel mailing list archives provide a huge choice of technical discussions (as well as flame wars) about preemptability, latency, priority inheritance and approaches to realtime support. It was fun to read back and watch how influential developers changed their minds over time. Especially Linus himself provides quite a few interesting quotes. In May 2002 he [stated](<http://marc.info/?l=linux-kernel&m=102229733006475&w=2>): 

With RTLinux, you have to split the app up into the "hard realtime" part (which ends up being in kernel space) and the "rest". 

Which is, in my opinion, the only sane way to handle hard realtime. No confusion about priority inversions, no crap. Clear borders between what is "has to happen _now_" and "this can do with the regular soft realtime". 

Four years later he said in a discussion about merging the realtime preemption patch during the Kernel Summit 2006: 

Controlling a laser with Linux is crazy, but everyone in this room is crazy in his own way. So if you want to use Linux to control an industrial welding laser, I have no problem with your using PREEMPT_RT. 

Equally interesting is his [statement](<http://lwn.net/Articles/178258/>) about priority inheritance in a huge discussion about realtime approaches in December 2005: 

Friends don't let friends use priority inheritance. Just don't do it. If you really need it, your system is broken anyway. 

Linus's clear statement that he wouldn't merge any PI code ever was rendered _ad absurdum_ when he merged the PI support for pthread_mutexes without a single comment only half a year later. 

Both are pretty good examples of the pragmatic approach of the Linux kernel development community and its key figures. Linus especially has always silently followed the famous words of the former German chancellor Konrad Adenauer: ""Why should I care about my chatter from yesterday? Nothing prevents me from becoming wiser."" 

#### Adding realtime response to the kernel

But back to the micro/nano-kernel versus in-kernel approaches which emerged in the late 90es. From both camps emerged commercial products and, more or less, active open source communities, but none of those efforts was commercially sustainable or ever got close to being merged into the official mainline kernel code base due to various reasons. Let me look at some of those reasons: 

  * Intrusiveness and maintainability: Most of those approaches lacked - and still lack - proper abstractions and smooth integration into the Linux kernel code base. `#ifdef`'s sprinkled all over the place are neither an incentive for kernel developers to delve into the code nor are they suitable for long-term maintenance. 

  * Complexity of usage: Dual-kernel approaches tend to be hard to understand for application programmers, who often have a hard time coping with a single API. Add a second API and the often backwards-implemented IPC mechanisms between the domains and failure is predictable. 

I'm not saying that it can't be done, it's just not suitable for the average programmer. 

  * Incompleteness: Some of those research approaches solve only parts of the problem, as this was their particular area of interest. But that prevents them from becoming useful in practice. 

  * Lack of interest: Some of the projects never made any attempt to approach the Linux kernel community, so the question of inclusion, or even partial merging of infrastructure, never came up. 

In October 2004, the real time topic got new vigor on the Linux kernel mailing list. MontaVista had integrated the results of research at the University of the German Federal Armed Forces at Munich into the kernel, replacing spinlocks with priority-inheritance-enabled mutexes. This posting resulted in one of the lengthiest discussions about realtime on the Linux kernel mailing list as almost everyone involved in efforts to solve the realtime problem surfaced and praised the superiority of their own approach. Interestingly enough, nobody from the academic camp participated in this heated argument. 

A few days after the flame fest started, the discussion was driven to a new level by kernel developer Ingo Molnar, who, instead of spending time with rhetoric, had implemented a different patch which, despite being clumsy and incomplete, built the starting point for the current realtime preemption patch. In no time quite a few developers interested in realtime joined Ingo's effort and brought the patch to a point which allowed real-world deployment within two years. During that time a huge number of interesting problems had to be solved: efficient priority inheritance, solving per cpu assumptions, preemptible RCU, high resolution timers, interrupt threading etc. and, as a further burden, the fallout from sloppily-implemented locking schemes in all areas across the kernel. 

#### Help from academia?

Those two years were mostly spent with grunt work and twisting our brains around hard-to-understand and hard-to-solve locking and preemption problems. No time was left for theory and research. When the dust settled a bit and we started to feed parts of the realtime patch to the mainline, we actually spent some time reading papers and trying to leverage the academic research results. 

Let me pick out priority inheritance and have a look at how the code evolved and why we ended up with the current implementation. The first version which was in Ingo's patchset was a rather simple approach with long-held locks, deep lock nesting and other ugliness. While it was correct and helped us to go forward it was clear that the code had to be replaced at some point. 

A first starting point for getting a better implementation was of course reading through academic papers. First I was overwhelmed by the sheer amount of material and puzzled by the various interesting approaches to avoid priority inversion. But, the more papers I read, the more frustrated I got. Lots of theory, proof-of-concept implementations written in Ada, micro improvements to previous papers, you all know the academic drill. I'm not at all saying that it was waste of time as it gave me a pretty good impression of the pitfalls and limitations which are expected in a non-priority-based scheduling environment, but I have to admit that it didn't help me to solve my real world problem either. 

The code was rewritten by Ingo Molnar, Esben Nielsen, Steven Rostedt and myself several times until we settled on the current version. The way led from the classic lock-chain walk with instant priority boosting through a scheduler-driven approach, then back to the lock-chain walk as it turned out to be the most robust, scalable and efficient way to solve the problem. My favorite implementation, though, would have been based on proxy execution, which already existed in Doug Niehaus's Kansas University Real Time project at that time, but unfortunately it lacked SMP support. Interestingly enough, we are looking into it again as non-priority-based scheduling algorithms are knocking at the kernel's door. But in hindsight I really regret that nobody--including myself--ever thought about documenting the various algorithms we tried, the up- and down-sides, the test results and related material. 

So it seems that there is the reverse problem on the real world developer side: we are solving problems, comparing and contrasting approaches and implementations, but we are either too lazy or too busy to sit down and write a proper paper about it. And of course we believe that it is all documented in the different patch versions and in the maze of the Linux kernel mailing list archives which are freely available for the interested reader. 

Indeed it might be a worthwhile exercise to go back and extract the information and document it, but in my case this probably has to wait until I go into retirement, and even then I fear that I have more favorable items on my ever growing list of things which I want to investigate. On the other hand, it might be an interesting student project to do a proper analysis and documentation on which further research could be based. 

#### On the value of academic research

I do not consider myself in any way to be representative of the kernel developer community, so I asked around to learn who was actually influenced by research results when working on the realtime preemption patch. Sorry for you folks, the bad news is that most developers consider reading research results not to be a helpful and worthwhile exercise in order to get real work done. The question arises why? Is academic OS research useless in general? Not at all. It's just incredibly hard to leverage. There are various reasons for this and I'm going to pick out some of them. 

First of all--and I have complained about this before--it's often hard to get access to papers because they are hidden away behind IEEE's paywall. While dealing with IEEE, a fact of life for the academic world, I personally consider it as a modern form of robber barony where tax payers have to pay for work which was funded by tax money in the first place. There is another problem I have with the IEEE monopoly. Universities' rankings are influenced by the number of papers written by their members and accepted at a IEEE conferences, which I consider to be one of the most idiotic quality measurement rules on the planet. And it's not only my personal opinion; it's also provable. 

I actually took the time to spend a day at a university where I could gain access to IEEE papers without wasting my private money. I picked out twenty recent realtime related papers and did a quick survey. Twelve of the papers were a rehash of well-known and well-researched topics, and at least half of them were badly written as well. From the remaining eight papers, six were micro improvements based on previous papers where I had a hard time figuring out why the papers had been written at all. One of those was merely describing the effects of converting a constant which influences resource partitioning into a runtime configurable variable. So that left two papers which seemed actually worthwhile to read in detail. Funny enough, I had already read one of those papers as it was publicly accessible in a slightly modified form. 

That survey really convinced me to stay away from IEEE forever and to consider the university ranking system even more suspicious. 

There are plenty of other sources where research papers can be accessed, but unfortunately the signal-to-noise ratio there is not significantly better. I have no idea how researchers filter that, but on the other hand most people wonder how kernel developers filter out the interesting stuff from the Linux kernel mailing list flood. 

One interesting thing I noticed while skimming through paper titles and abstracts is that the Linux kernel seems to have become the most popular research vehicle. On one site I found roughly 600 Linux-based realtime and scheduling papers which were written in the last 18 months. About 10% of them utilized the realtime preemption patch as their baseline operating system. Unfortunately almost none of the results ever trickled through to the kernel development community, not to mention actually working code being submitted to the Linux kernel mailing list. 

As a side note: one paper even mentioned a hard-to-trigger longstanding bug in the kernel which the authors fixed during their research. It took me some time to map the bug to the kernel code, but I found out that it got fixed in the mainline about three months after the paper was published--which is a full kernel release cycle. The fix was not related to this research work in any way, it just happened that some unrelated changes made the race window wider and therefore made the bug surface. I was a bit grumpy when I discovered this, but all I can ask for is: please send out at least a description of a bug you trip over in your research work to the kernel community. 

Another reason why it's hard for us to leverage research results is that academic operating system research has, as probably any other academic research area, a few interesting properties: 

  * **Base concepts in research are often several decades old** , but they don't show up in the real world even if they would be helpful to solve problems which have been worked around for at least the same number of decades more or less. 

We discussed the sporadic server model yesterday at OSPERT, but it has been around for 27 years. I assume that hundreds of papers have been written about it, hundreds of researchers and students have improved the details, created variations, but there is almost no operating system providing support for it. As far as I know Apple's OSX is the only operating system which has a scheduling policy which is not based on priorities but, as I learned, it's well hidden away from the application programmer. 

  * **Research often happens on narrow aspects of an already narrow problem space**. That's understandable as you often need to verify and contrast algorithms on their own merit without looking at other factors. But that leaves the interested reader like me with a large amount of puzzle pieces to chase and fit together, which often enough made me give up. 

  * **Research often happens on artificial application scenarios**. While again understandable from the research point of view, it makes it extremely hard, most of the time, to expand the research results into generalized application scenarios without shooting yourself in the foot and without either spending endless time or giving up. I know that it's our fault that we do not provide real application scenarios to the researchers, but in our defense I have to say that in most of the cases we don't know what downstream users are actually doing. We only get a faint idea of it when they complain about the kernel not doing what they expect. 

  * **Research often tries to solve yesterday's problems over and over** while the reality of hardware and requirements have already moved to the next levels of complexity. I can understand that there are still interesting problems to solve, but seeing the gazillionst paper about priority ceilings on uniprocessor systems is not really helpful when we are struggling with schedulability, lock scaling and other challenges on 64- (and more) core machines. 

  * **Comparing and contrasting research results is almost impossible**. Even if a lot of research happens on Linux there is no way to compare and contrast the results as researchers, most of the time, base their work on completely different base kernel versions. We talked about this last year and I have to admit that neither Peter nor myself found enough spare time to come up with an approach to create a framework on which the various research groups could base their scheduler of the day. We haven't forgotten about this, but while researchers have to write papers, we get our time occupied by other duties. 

  * **Research and education seem to happen in different universes**. It seems that operating system and realtime research have little influence on the education of Joe Average Programmer. I'm always dumbstruck when talking to application programmers who have not the faintest idea of resources and their limitations. It seems that the resource problems on their side are all solvable by visiting the hardware shop across the street and buying the next-generation machine. That approach also manifests itself pretty well in the "enterprise realtime" space where people send us test cases which refuse to even start on anything smaller than a machine equipped with 32GB of RAM and at least 16 cores. 

If you have any chance to influence that, then please help to plant at least some clue on the folks who are going to use the systems you and we create. 

A related observation is the inability of hardware and software engineers to talk to each other when a system is designed. While I observe that disconnect mainly on the industry side, I have the feeling that it is largely true in the universities as well. No idea how to address this issue, but it's going to be more important the more the complexity of systems increases. 

I'll stop bashing on you folks now, but I think that there are valid questions and we need to figure out answers to them if we want to get out of the historically grown state of affairs someday. 

#### In conclusion

We are happy that you use Linux and its extensions for your research, but we would be even more happy if we could deal with the outcome of your work in an easier way. In the last couple of years we started to close the gap between researchers and the Linux kernel community at OSPERT and at the Realtime Linux Workshop and I want to say thanks to Stefan Petters, Jim Anderson, Gerhard Fohler, Peter Zijlstra and everyone else involved. It's really worthwhile to discuss the problems we face with the research community and we hope that you get some insight into the problems we face and requirements which are behind our pragmatic approach to solve them. 

And of course we appreciate that some code which comes out straight of the research laboratory (the [EDF scheduler](<http://lwn.net/Articles/356576/>) from [ReTiS](<http://retis.sssup.it/>), Pisa) actually got cleaned up and published on the Linux kernel mailing list for public discussion and I really hope that we are going to see more like this in the foreseeable future. Problem complexity is increasing, unfortunately, and we need all the collective brain power to address next year's challenges. We already started the discussion and first interesting patches have shown up, so really I hope we can follow down that road and get the best out of it for all of us. 

Thanks for your attention. 

#### Feedback

I got quite a bit of feedback after the talk. Let me answer some of the questions. 

**Q:** Is there any place outside LKML where discussion between academic folks and the kernel community can take place? 

**A:** Bjorn Brandenberg suggested setting up a mailing list for research related questions, so that the academics are not forced to wade through the LKML noise. If a topic needs a broader audience we always can move it to LKML. I'm already working on that. It's going to be low traffic, so you should not be swamped in mail. 

**Q:** Where can I get more information about the realtime preemption patch ? 

**A:** General information can be found on [the realtime Linux wiki](<https://rt.wiki.kernel.org>), [this LWN article](<http://lwn.net/Articles/146861/>), and [this Linux Symposium paper [PDF]](<http://www.linuxsymposium.org/archives/OLS/Reprints-2007/rostedt-Reprint.pdf>). 

**Q** : Which technologies in the mainline Linux kernel emerged from the realtime preemption patch? 

**A:** The list includes: 

  * the Generic interrupt handling framework. See: Linux/Documentation/DocBook/genericirq and [this LWN article](<http://lwn.net/Articles/184750/>). 

  * Threaded interrupt handlers, [described in LWN](<http://lwn.net/Articles/302043/>) and [again in LWN](<http://lwn.net/Articles/321663/>). 

  * The mutex infrastructure. See: Linux/Documentation/mutex-design.txt 

  * High-resolution timers, including NOHZ idle support. See: Linux/Documentation/timers/highres.txt and [these presentation slides](<http://www.kernel.org/pub/linux/kernel/people/tglx/hrtimers/ols2006-hrtimers.pdf>). 

  * Priority inheritance support for user space pthread_mutexes. See: Linux/Documentation/pi-futex.txt, Linux/Documentation/rt-mutex.txt, Linux/Documentation/rt-mutex-design.txt, [this LWN article](<http://lwn.net/Articles/360699/>), and [this Realtime Linux Workshop paper [PDF]](<http://lwn.net/images/conf/rtlws11/papers/proc/p10.pdf>). 

  * Robustness support for user-space pthread_mutexes. See: Linux/Documentation/robust-futexes.txt and [this LWN article](<http://lwn.net/Articles/172149/>). 

  * The lock dependency validator, [described in LWN](<http://lwn.net/Articles/185666/>). 

  * The kernel tracing infrastructure, as described in a series of LWN articles: [1](<http://lwn.net/Articles/322666/>), [2](<http://lwn.net/Articles/365835/>), [3](<http://lwn.net/Articles/366796/>), and [4](<http://lwn.net/Articles/370423/>). 

  * Preemptible and hierarchical RCU, also documented in LWN: [1](<http://lwn.net/Articles/253651/>), [2](<http://lwn.net/Articles/220677/>), [3](<http://lwn.net/Articles/371986/>), and [4](<http://lwn.net/Articles/305782/>). 

**Q:** Where do I get information about the Realtime Linux Workshop? 

**A:** The 2010 realtime Linux Workshop (RTLWS) will be in Nairobi, Kenya, Oct. 25-27th. The 2011 RTLWS is planned to be at Kansas University (not confirmed yet). Further information can be found on [the RTLWS web page](<http://www.osadl.org/RTLWS-2010.rtlws-2010.0.html>). General information about the organisation behind RTLWS can be found on [the OSADL page](<http://www.osadl.org>), and information about it's academic members is on [this page](<http://www.osadl.org/Academic-Members.osadl_academic_members.0.html>). 

#### Conference impressions

I stayed for the main conference, so let me share my impressions. First off the conference was well organized and, in general, the atmosphere was not really different from an open source conference. The realtime researchers seem to be a well-connected and open-minded community. While they take their research seriously, at least most of them admit freely that the ivory tower they are living in can be a complete different universe. This was pretty much observable in various talks where the number of assumptions and the perfectly working abstract hardware models made it hard for me to figure out how the results of this work could be applied to reality. 

The really outstanding talks were the keynotes on day two and three. 

On Thursday, Norbert When from the Technical University Kaiserslautern gave an interesting talk titled [Hardware modeling: A critical assessment with case studies [PDF]](<http://ecrts.eit.uni-kl.de/fileadmin/user_media/ecrts10/Keynote_ECTRS_wehn.pdf>). Norbert is working on hardware modeling and low-level software for embedded devices, so he is not the typical speaker you would expect at a realtime-focused conference. But it seems that the program committee tried to bring some reality into the picture. Norbert gave an impressive overview over the evolution of hardware and the reasons why we have to deal with multi-core hardware and have to face the fact that today's hardware is not designed for predictability and reliability. So realtime folks need to rethink their abstract models and take more complex aspects of the overall system into account. 

One of the interesting aspects was his view on energy efficient computing: A cloud of 1.7 million AMD Opteron cores consumes 179MW while a cloud of 10 million Xtensa cores provides the same computing power at 3MW. Another aspect of power-aware computing is the increasing role of heterogeneous systems. Dedicated hardware for video decoding is about 100 times more power efficient than a software-based solution on a general-purpose CPU. Even specialized DSPs consume about 10 times more power for the same task than the optimized hardware solution. 

But power optimized hardware has a tradeoff: the loss of flexibility which is provided by software. But the mobile space has already arrived in the heterogeneous world, and researchers need to become aware of the increased complexity to analyze such hybrid constructs and develop new models to allow the verification of these systems in the hardware design phase. Workarounds for hardware design failures in application specific systems are orders of magnitudes more complex than on general purpose hardware. All in all, he gave his colleagues from the operating system and realtime research communities quite a list of homework assignments and connected them back to earth. 

The Friday morning keynote was a surprising reality check as well. Sanjoy Baruah from the University of North Carolina at Chapel Hill titled his talk "Why realtime scheduling theory still matters". Given the title one would assume that the talk would be focused on justifying the existence of the ivory tower, but Sanjoy was very clear about the fact that the realtime and scheduling research has focused for too long on uniprocessor systems and is missing answers to the challenges of the already-arrived multi-core era. He gave pretty clear guidelines about which areas research should focus on to prove that it still matters. 

In addition to the classic problem space of verifiable safety-critical systems, he was calling for research which is relevant to the problem space and built on proper abstractions with a clear focus on multi-core systems. Multi-core systems bring new--and mostly unresearched--challenges like mixed criticalities, which means that safety critical, mission critical and non critical applications run on the same system. All of them have different requirements with regard to meeting their deadlines, resource constraints, etc., and therefore bring a new dimension into the verification problem space. Other areas which need care, according to Sanjoy, are component-based designs and power awareness. 

It was good to hear that despite our usual perception of the ivory tower those folks have a strong sense of reality, but it seems they need a more or less gentle reminder from time to time. ECRTS was a real worthwhile conference and I can only encourage developers to attend such research-focused events and keep the communication and discussion between our perceived reality and the not-so-disconnected other universe alive.
