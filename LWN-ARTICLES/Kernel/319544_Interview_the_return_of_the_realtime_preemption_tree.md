---
title: "Interview: the return of the realtime preemption tree"
url: https://lwn.net/Articles/319544/
date: "February 16, 2009"
category: Preemption; Realtime
author: "By Jonathan Corbet February 16, 2009"
---

By **Jonathan Corbet**  
February 16, 2009

The realtime preemption project is a longstanding effort to provide deterministic response times in a general-purpose kernel. Much code resulting from this work has been merged into the mainline kernel over the last few years, and a number of vendors are shipping commercial products based upon it. But, for the last year or so, progress toward getting the rest of the realtime work into the mainline has slowed. 

On February 11, realtime developers Thomas Gleixner and Ingo Molnar resurfaced with the announcement of [a new realtime preemption tree](<http://lwn.net/Articles/318949/>) and a newly reinvigorated development effort. Your editor asked them if they would be willing to answer a few questions about this work; their response went well beyond the call of duty. Read on for a detailed look at where the realtime preemption tree stands and what's likely to happen in the near future. 

**LWN** : The 2.6.29-rc4-rt1 announcement notes that you're coming off a 1.5-year sabbatical. Why did you step away from the RT patches so long; have you been hanging out on the beach in the mean time? :) 

**Thomas** : We spent a marvelous time at the x86 lagoon, a place with an extreme contrast of antiquities and modern art. :) 

Seriously, we underestimated the amount of work which was necessary to bring the unified x86 architecture into shape. Nothing to complain about; it definitely was and still is a worthwhile effort and I would not hesitate longer than a fraction of a second to do it again. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

**Ingo** : Yeah, hanging out on the beach for almost two years was well-deserved for both of us. We met Linus there and it was all fun and laughter, with free beach cocktails, pretty sunsets and camp fires. [ All paid for by the nice folks from Microsoft btw., - those guys sure know how to please a Linux kernel hacker! ;-) ] 

So what has brought you back to the realtime work at this time? 

**Thomas** : Boredom and nostalgia :) In fact I never lost track of the real time work since we took over x86 maintenance, but my effort was restricted to decode hard to solve problems and make sure that the patches were kept in a usable shape. Right now I have the feeling that we need to put more development effort into preempt-rt again to keep its upstream visibility and make progress on merging the remaining parts. 

The most important reason for returning was of course our editors challenge in [The Grumpy Editor's guide to 2009](<http://lwn.net/Articles/313045>): "The realtime patch set will be mostly merged by the end of the year..." 

**Ingo** : When we left for the x86 land more than 1.5 years ago, the -rt patch-queue was a huge pile of patches that changed hundreds of critical kernel files and introduced/touched ten thousand new lines of code. Fast-forward 1.5 years and the -rt patchqueue is a humungous pile of patches that changes nearly a thousand critical kernel files and introduces/touches twenty-thirty thousand lines of code. So we thought that while the project is growing nicely, it is useful and obviously people love it - the direction of growth was a bit off and that this particular area needs some help. 

Initially it started as a thought experiment of ours: how much time and effort would it take to port the most stable -rt patch (.26-rt15) to the .29-tip tree and could we get it to boot? Turns out we are very poor at thought experiments (just like we are pretty bad at keeping patch queues small), so we had to go and settle the argument via some hands-on hacking. Porting the queue was serious fun, it even booted after a few dozen fixes, and the result was the .29-rt1 release. 

Maintaining the x86 tree for such a long time and doing many difficult conceptual modernizations in that area was also very helpful when porting the -rt patch-queue to latest mainline. 

Most of the code it touched and most of the conflicts that came up looked strangely familiar to us, as if those upstream changes went through our trees =;-) 

(It's certainly nothing compared to the beach experience though, so we are still looking at returning for a few months to a Hawaii cruise.) 

How well does the realtime code work at this point? What do you think are the largest remaining issues to be tackled? 

**Thomas** : The realtime code has reached quite a stable state. The 2.6.24/26 based versions can definitely be considered production ready. I spent a lot of time to sort out a huge amount of details in those versions to make them production stable. Still we need to refactor a lot of the patches and look for mainline acceptable solutions for some of the real time related changes. 

**Ingo** : To me what settled quite a bit of "do we need -rt in mainline" questions were the spin-mutex enhancements it got. Prior that there were a handful of pretty pathologic workload scenarios where -rt performance tanked over mainline. With that it's all pretty comparable. 

The patch splitup and patch quality has improved too, and the queue we ported actually builds and boots at just about every bisection point, so it's pretty usable. A fair deal of patches fell out of the .26 queue because they went upstream meanwhile: tracing patches, scheduler patches, dyntick/hrtimer patches, etc. 

It all looks a lot less scary now than it looked 1.5 years ago - albeit the total size is still considerable, so there's definitely still a ton of work with it. 

What are your current thoughts with regard to merging this work into the mainline? 

**Thomas** : First of all we want to integrate the -rt patches into our -tip git repository which makes it easier to keep -rt in sync with the ongoing mainline development. The next steps are to gradually refactor the patches either by rewriting or preferably by pulling in the work which was done in Stevens git-rt tree, split out parts which are ready and merge them upstream step by step. 

**Ingo** : IMO the key thought here is to move the -rt tree 'ahead of the upstream development curve' again, and to make it the frontier of Linux R&D. With a 2.6.26 basis that was arguably hard to do. With a partly-2.6.30 basis (which the -tip tree really is) it's a lot more ahead of the curve, and there are a lot more opportunities to merge -rt bits into upstream bits wherever there's accidental upstream activity that we could hang -rt related cleanups and changes onto. We jumped almost 4 full kernel releases, that moves -rt across 1 year worth of upstream development - and keeps it at that leading edge. 

Another factor is that most of the top -rt contributors are also -tip contributors so there's strong synergy. 

The -tip tree also undergoes serious automated stabilization and productization efforts, so it's a good basis for development _and_ for practical daily use. For example there were no build failures reported against .29-rt1, and most of the other failures that were reported were non-fatal as well and were quickly fixed. One of the main things we learned in the past 1.5 years was how to keep a tree stable against a wild, dangerous looking flux of modifications. 

YMMV ;-) 

Thomas once told me about a scheme to patch rtmutex locks into/out of the kernel at boot time, allowing distributors to ship a single kernel which can run in either realtime or "normal" mode. Is that still something that you're working on? 

**Thomas** : We are not working on that right now, but it is still on the list of things which need to be investigated. 

**Ingo** : That still sounds like an interesting feature, but it's pretty hard to pull it off. We used to have something rather close to that, a few years ago: a runtime switch that turned the rtmutex code back into spinning code. It was fragile and hard to maintain and eventually we dropped it. 

Ideally it should be done not at boot time but runtime - via the stop-machine-run mechanism or so. [extended perhaps with hibernation bits that force each task into hitting user-mode, so that all locks in the system are released] 

It's really hard to implement it, and it is definitely not for the faint hearted. 

The RT-preempt code would appear to be one of the biggest exceptions to the "upstream first" rule, which urges code to be merged into the mainline before being shipped to customers. How has that worked out in this case? Are there times when it is good to keep shipping code out of the mainline for such a long time? 

**Thomas** : It is an exception which was only acceptable because preempt-rt does not introduce new user space APIs. It just changes the run time behaviour of the kernel to a deterministic mode. 

All changes which are user space API related (e.g. PI futexes) were merged into mainline before they got shipped to customers via preempt-rt and all bug fixes and improvements of mainline code were sent upstream immediately. Preempt-rt was never a detached project which did not care about mainline. 

When we started preempt-rt there was huge demand on the customer side \- both enterprise and embedded - for an in kernel realtime solution. The dual kernel approaches of RTAI, RT-Linux and Xenomai had no chance to get ever accepted into the mainline and the handling of the dual kernel environment has never been an easy task. With preempt-rt you just switch the kernel under a stock mainline user space environment and voila your application behaves as you would expect - most of the time :) Dual kernel environments require different libraries, different APIs and you can not run the same binary on a non -rt enabled kernel. Debugging preempt-rt based real time applications is exactly the same as debugging non real time applications. 

[PULL QUOTE:  While we never had doubts that it would be possible to turn Linux into a real time OS, it was clear from the very beginning that it would be a long way until the last bits and pieces got merged.  END QUOTE] While we never had doubts that it would be possible to turn Linux into a real time OS, it was clear from the very beginning that it would be a long way until the last bits and pieces got merged. The first question Ingo asked me when I contacted him in the early days of preempt-rt was: "Are you sure that you want to touch every part of the kernel while working on preempt-rt?". This question was absolutely legitimate; in the first days of preempt-rt we really touched every part of the kernel due to problems which were mostly locking and preemption related. The fixes have been merged upstream and especially in the locking area we got a huge improvement in mainline due to lock debugging, conversion to mutexes, etc. and a general better awareness of locking and preemption semantics. 

preempt-rt was always a great breeding ground for fundamental changes in the kernel and so far quite a large part of the preempt-rt development has been integrated into the mainline: PI-futexes, high-resolution timers ... I hope we can keep that up and provide soon more interesting technological changes which emerged originally from the preempt-rt efforts. 

**Ingo** : Preempt-rt turns the kernel's scheduling, lock handling and interrupt handling code upside down, so there was no realistic way to merge it all upstream without having had some actual field feedback. It is also unique in that you need _all_ those changes to have the new kernel behavior - there's no real gradual approach to the -rt concept itself. That adds up to a bit of a catch-22: you don't get it upstream without field use, and you don't get field use without it being upstream. 

Deterministic execution is a major niche, one of which was not effectively covered by the mainstream kernel before. It's perhaps the last major technological niches in existence that the stock upstream kernel does not handle yet, and it's no wonder that the last one out is in that precise position for conceptually hard reasons. 

In short: all the easy technologies are upstream already ;-) 

Nevertheless we strictly got all user-ABI changes upstream first: PI-futexes in particular. The rest of -rt is "just" a new kernel option that magically turns kernel execution into deterministic mode. 

Where would be the best starting point for a developer who wishes to contribute to this effort? 

**Thomas** : Nothing special with the realtime patches. Just kernel development as usual: get the patches, apply them, run them on your machine and test. If problems arise, provide bug reports or try to fix them yourself and send patches. Read through the code and start providing improvements, cleanups ... 

**Ingo** : Beyond the "try it yourself, follow the discussions, and go wherever your heart tells you to go" suggestion, there's a few random areas that might need more attention: 

  * Big Kernel Lock removal. It's critical for -rt. We still have the tip:core/kill-the-BKL branch, and if someone is interested it would be nice to drive that effort forward. A lot of nice help-zap-the-BKL patches went upstream recently (such as the device-open patches), so we are in a pretty good position to try the kill-the-BKL final hammer approach too. 

[I have just done a (raw!) refresh and conflict resolution merge of that tree to v2.6.29-rc5. Interested people can find it at: 

```
git pull \
                git://git.kernel.org/pub/scm/linux/kernel/git/tip/linux-2.6-tip.git \
                core/kill-the-BKL
```

Warning: it might not even build. ] 

  * Look at Steve's git-rt tree and split out and gradually merge bits. A fair deal of stuff has been cleaned up there and it would be nice to preserve that work. 

  * Latency measurements and tooling. Go try the latency tracer, the function graph tracer and ftrace in general. Try to find delays in apps caused by the kernel (or caused by the app itself), and think about whether the kernel's built-in tools could be improved. 

  * Try Thomas's cyclictest utility and try to trace and improve those worst-case latencies. A nice target would be to push the worst-case latencies on a contemporary PC below 10 microseconds. We were down to about 13 microseconds with a hack that threaded the timer IRQ with .29-rt1, so it's possible to go below 10 microseconds i think. 

  * And of course: just try to improve the mainline kernel - that will improve the -rt kernel too, by definition :-) 

But as usual, follow your own path. Independent, critical thinking is a lot more valuable than follow-the-crowd behavior. [As long as it ends up producing patches (not flamewars) that is ;-)] 

And by all means, start small and seek feedback on lkml early and often. Being a good and useful kernel developer is not an attribute but a process, and good processes always need time, many gradual steps and a feedback loop to thrive. 

Many thanks to Thomas and Ingo for taking the time to answer (in detail!) this long list of questions.
