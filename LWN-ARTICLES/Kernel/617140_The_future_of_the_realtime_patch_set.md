---
title: The future of the realtime patch set
url: https://lwn.net/Articles/617140/
date: "October 21, 2014"
category: Realtime
author: "By Jake Edge October 21, 2014 Linux Plumbers Conference"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
October 21, 2014

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2014-Linux_Plumbers_Conference>)

In a followup to last year's [report on the future of realtime Linux](<https://lwn.net/Articles/572740/>), Thomas Gleixner once again summarized the status of the long-running patch set. The intervening year did not result in the industry stepping up to fund further work, which led Gleixner to declare that realtime Linux is now just his hobby. That means new releases will be done as his time allows and may eventually lead to dropping the patch set altogether if the widening gap between mainline and realtime grows too large. 

Gleixner was speaking at the realtime microconference that was part of this year's [Linux Plumbers Conference](<http://www.linuxplumbersconf.org/2014/>) (LPC). Much of what he had to say was a reprise from earlier in the week when the [Real Time Linux Workshop](<https://www.osadl.org/Dusseldorf-2014.rtlws16-dusseldorf-2014.0.html>) (RTLWS) was held in Düsseldorf, Germany at the same venue as LPC. In effect, his October 16 LPC talk was a recap of the discussion from RTLWS. 

#### Status

[ ![\[Thomas Gleixner\]](https://static.lwn.net/images/2014/lpc-gleixner-sm.jpg) ](<https://lwn.net/Articles/617256/>)

He started with a brief report on the status of the patches themselves, along with what needs to be worked on to move more of the patch set into the mainline. At RTLWS, they came up with some "easy things to go upstream", including some code that avoids the "complexities of workqueues", he said. The discussion also resulted in some plans to change some of the patches from last year so they can go upstream as well. 

Maintaining the patch set takes a lot of time and effort. But it has also led to improving the quality of the mainline kernel over time. Another effort could lead to similar results, he said: annotating the various uses of the `preempt_disable()` call. Those calls are made to protect some kernel data structure, but it is not at all clear what they are protecting. That turns them into a "per-CPU BKL [big kernel lock] that is painful to understand". 

For the realtime patches, sections where preemption is disabled need to be made as small as possible, but that is difficult to do if it is unclear what is being protected. Currently, developers creating patches for performance improvements often disable preemption, but do not care that it has an impact on the out-of-tree realtime patch set. It is likely, though, that annotating those uses in the mainline will not only help realtime, it will find bugs of various kinds as well. 

People have asked over the years how they can help with realtime, Gleixner said, but there isn't a simple answer to that. He cannot just assign people to work on certain pieces. Instead, interested people need to find some part they are interested in, understand it, and then start sending patches and handling the review comments that are generated. It is a matter of new people catching up "with the knowledge of people doing it for ten years". 

In many cases, adding realtime patches to the mainline is a matter of enabling certain features, then "see what explodes and then figure it out and fix it", he said. 

#### Success and disaster

Realtime Linux is both a success and a disaster, Gleixner said. It has proven that it is possible to turn a general-purpose operating system (OS) like Linux into a realtime OS, which was something that many academics said couldn't be done. Ideas from Doug Niehaus in the 1990s ended up in the realtime patches ten years later, though there are still some of those ideas, like proxy locking, that have not yet been tried. 

It is also a success because it has made the Linux mainline into "a better place", he said. There are a number of features that came from realtime and made their way into the mainline, including high-resolution timers, the foundations of the "nohz" (tickless) patches, threaded interrupt handlers, generic timekeeping, real mutexes, and so on. In addition, the "lockdep" locking validator has helped many kernel developers understand how locking should be done. 

On the disaster side, though, is the lack of funding to actually complete the project. There has been a lot of work put into getting the realtime patches upstream. That is partly the "fault" of the US Navy, which noted in the contract that started the realtime effort that all of the work should be upstreamed on a "best effort basis". But projects go away and budgets dry up, Gleixner said. 

In five years, the realtime project has gone from a "moderately lively" community-style project to roughly one-and-a-half people working just to keep it alive. The maintenance of the patch set is "not fun", with its six-month development cycle, followed by six months of stabilization. Now the funding has essentially "dried up to zero". 

Earlier in the microconference, there had been several indications of use cases where realtime will be used in products. But there is (seemingly) no interest from the industry to keep the project alive. That is "disturbing", Gleixner said. 

There are a number of "well-funded and vibrant projects" out there, but there are others that have no funding even when they are widely used. He pointed to projects like Apache and the TYPO3 content-management system as examples of the former. Those kinds of projects are used by new companies that are interested in keeping the projects alive because it is part of their business model. 

On the other hand, the typical realtime use cases, such as embedded, automotive, and automation, are from companies that build machines. So the hardware is where they think the value is, Gleixner said. Lots of the value is actually in the software, but companies that do "mechanical stuff" often don't see that. Or at least that is his theory. 

The Linux Foundation (LF) and the [Open Source Automation Development Lab](<https://www.osadl.org/>) (OSADL, which organizes RTLWS each year) have both been trying to drum up funds for the last year or more without success. They are trying to get companies to collaboratively provide money or capable developers. There was some tentative interest, but nothing really came of it, he said. He wondered if the industry was treating it like the "Mikado game", where "whoever moves first, loses". 

#### Realtime as a hobby

So, he has concluded that he will be working on the realtime patches on a "hobbyist basis". He needs to work on things that bring in money for him and his family; the realtime work evidently does not fit the bill. He will be working on the 3.16-rt patches in his hobby time, though he noted that he does have other hobbies too. 

He has received email from around two dozen companies asking for a schedule for the 3.16-rt release because they need to make product plans. He has responded by telling them that there is no budget and asking if they want to help. He got different types of responses, he said. Some didn't reply at all, which he called "honest". Others said they had no money. It is "crazy" to build a product around a technology and to have no budget to work on it, he said. Others said they would be building their products around the 3.12-rt kernel. 

While it is a hobby project as of now, that can change, but it is up to those who want to see that happen to make it happen. The LF and OSADL are still working on it, so interested parties should contact them. 

He also noted that if funding takes too long to materialize, the realtime project will be harder to restart. There is a gap between mainline and realtime that grows over time. The scalability and performance improvements flowing into mainline are "massive", he said. Those developers often come up with solutions that are "horrible for realtime". Since the realtime patches are out of the mainline, there is nothing to stop those kinds of changes being made. 

There are both technical and political problems to be solved before the remaining realtime patches can be merged into the mainline. Both types of problem can be solved, Gleixner said. Linus Torvalds is willing to merge realtime patches, as long as they don't interfere with the rest of the kernel core. But, before he does so, he also wants to see some organization with a long-term interest in supporting and maintaining the realtime code. 

Patches for realtime are quite different than some random driver or filesystem; those can be removed or ignored if they stop being maintained. But realtime is "a different beast" as it changes the core kernel code. It also "causes kernel developers to think more", which is a good thing, he said. 

The industry appears not to have heard the call from a year ago, so it may be optimistic to think that will change now. The future of realtime Linux may be fairly bleak. It will be unfortunate if things continue down their current path and companies realize in a year or two that they need the hardware support or other features from a 4.4 kernel, say, but also need the latency guarantees that the realtime patches provide. It could quite possibly be far too expensive to try to pick up the project and move forward at that point. 

[ I would like to thank the Linux Foundation for travel assistance to Düsseldorf for LPC. ]
