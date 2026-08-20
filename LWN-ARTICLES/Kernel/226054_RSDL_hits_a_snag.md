---
title: RSDL hits a snag
url: https://lwn.net/Articles/226054/
date: "March 14, 2007"
category: Interactivity; Scheduler; Staircase scheduler
author: 
---

In [last week's episode](<http://lwn.net/Articles/224865/>), the Rotating Staircase Deadline Scheduler (RSDL) had appeared out of the blue and was busily impressing testers left and right. One person even called for it to go straight into 2.6.21. In reality, the replacement of something as fundamental as the CPU scheduler was never going to be an entirely smooth operation. So it's not all that surprising that the RSDL has run into an obstacle or two. 

The biggest snag would appear to be [this workload reported by Mike Galbraith](<https://lwn.net/Articles/226055/>). Mike is trying to run some CPU hogs (MP3 encoding, in particular) in the background while watching some interactive eye candy. It's a load that works with the current scheduler, but it becomes sluggish when running under RSDL. There have been a couple of other reports of a visible interactive slowdown when serious computation is going on - though others have [reported better results](<https://lwn.net/Articles/226056/>). 

There is little surprise in the appearance of behavioral regressions for certain workloads. Few people would have expected RSDL to be perfect within a week of its first posting. The real difficulty, instead, is that RSDL creator Con Kolivas has reacted in a somewhat defensive manner, [refusing](<https://lwn.net/Articles/226058/>) to see the behavior as a regression: 

Your expectations of what you should be able to do are simply skewed. Find what cpu balance you loved in the old one (and I believe it wasn't that much more cpu in favour of X if I recall correctly) and simply change the nice setting on your lame encoder \- since you're already setting one anyway. 

We simply cannot continue arguing that we should dish out unfairness in any manner any more. It will always come back and bite us where we don't want it. We are getting good interactive response with a fair scheduler yet you seem intent on overloading it to find fault with it. 

Con's position is that the scheduler should strive to provide fairness and low latency; any further expectations about interactive response should then be addressed by playing with nice levels. The interactivity estimator built into the current scheduler is just too difficult to work with; the kernel should not be in that particular business. The problem is that this approach conflicts with how Linux users have come to expect things to work. 

As soon as one looks at improving RSDL for these situations, one gets into the same old discussions on improving interactive response in general. Linus [pointed out](<https://lwn.net/Articles/226059/>) that RSDL's way of scheduling is not quite as fair as it could be, since it does not always account for work in the right place: 

And the problem is that a lot of clients actually end up doing *more* in the X server than they do themselves directly. Doing things like showing a line of text on the screen is a lot more expensive than just keeping track of that line of text, so you end up with the X server easily being marked as getting "too much" CPU time, and the clients as being starved for CPU time. And then you get bad interactive behaviour. 

There are a couple of ways of handling problems like this. One is to just favor the X server, either by somehow marking it as the core of interactive behavior or by simply raising its priority. Con has been in favor of the latter approach; to that end, he has posted [a separate patch](<https://lwn.net/Articles/226062/>) which is aimed at improving latencies for all processes, even when they are not all running at the same priority levels. There have not been any follow-up results reported as of this writing. 

This difficulty may well not keep RSDL out of the mainline kernel. The advantages inherent in dumping the interactivity heuristics are large, and RSDL does seem to improve life for a number of users. Noticeable performance regressions for some workloads are a problem, though; nobody wants to field a bunch of "2.6.x turned my response to crap" messages from unhappy users. So expect some iterations on this project yet - and, perhaps, an additional kernel cycle or two before it can be merged.
