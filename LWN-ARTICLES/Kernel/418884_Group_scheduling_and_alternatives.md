---
title: Group scheduling and alternatives
url: https://lwn.net/Articles/418884/
date: "December 6, 2010"
category: "Group scheduling; Scheduler-Group scheduling"
author: "By Jonathan Corbet December 6, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 6, 2010

The [TTY-based group scheduling](<https://lwn.net/Articles/415740/>) patch set has received a lot of discussion on LWN and elsewhere; some distributors are rushing out kernels with this code added, despite the fact that it has not yet been merged into the mainline. That patch has evolved slightly since it was last discussed here. There have also been some interesting conversations about alternatives; this article will attempt to bring things up to date. 

The main change to the TTY-based group scheduling patch set is that it is, in fact, no longer TTY-based. The identity of the controlling terminal was chosen as a heuristic which could be used to group together tasks which should compete with each other for CPU time, but other choices are possible. An obvious possibility is the session ID. This ID is used to identify distinct process groups; a process starts a new session with the `setsid()` system call. Since sessions are already used to group together related processes, it makes sense to use the session ID as the key when grouping processes for scheduling. More recent versions of the patch do exactly that. The session-based group scheduling mechanism appears to be stabilizing; chances are good that it will be merged in the 2.6.38 merge window. 

Meanwhile, there have been a couple of discussions led by vocal proponents of other approaches to interactive scheduling. It is fair to say that neither is likely to find its way into the mainline. Both are worth a look, though, as examples of how people are thinking about the problem. 

Colin Walters [asked](<https://lwn.net/Articles/418885/>) about whether group scheduling could be tied into the "niceness" priorities which have been implemented by Unix and Linux schedulers for decades. People are used to `nice`, he said, but they would like it to work better. Creating groups for nice levels would help to make that happen. But Linus was [not excited](<https://lwn.net/Articles/418739/>) about this idea; he claims that almost nobody uses `nice` now and that is unlikely to change. 

More to the point, though: the semantics implemented by `nice` are very different from those offered by group scheduling. The former is entirely priority-based, making the promise that processes with a higher "niceness" will get less processor time than those with lower values. Group scheduling, instead, is about isolation - keeping groups of processes from interfering with each other. The concept of priorities is poorly handled by group scheduling now, it's just not how that mechanism works. Group scheduling will not cause one set of processes to run in favor of another; it just ensures that the division of CPU time between the groups is fair. 

Colin went on to suggest that using groups would improve `nice`, giving the results that users really want. But changing something as fundamental as the effects of niceness would be, in a very real sense, an ABI change. There may not be many users of `nice`, but installations which depend on it would not appreciate a change in its semantics. So `nice` will stay the way it is, and group scheduling will be used to implement different (presumably better) semantics. 

The group scheduling discussion also featured [a rare appearance by Con Kolivas](<https://lwn.net/Articles/418887/>). Con's view is that the session-based group scheduling patch is another attempt to put interactivity heuristics into the kernel - an approach which has failed in the past: 

You want to program more intelligence in to work around these regressions, you'll just get yourself deeper and deeper into the same quagmire. The 'quick fix' you seek now is not something you should be defending so vehemently. The "I have a solution now" just doesn't make sense in this light. I for one do not welcome our new heuristic overlords. 

Con's alternative suggestion was to put control of interactivity more directly into the hands of user space. He would attach a parameter to every process describing its latency needs. Applications could then be coded to communicate their needs to the kernel; an audio processing application would request the lowest latency, while `make` would inform the kernel that latency matters little. Con would also add a global knob controlling whether low-latency processes would also get more CPU time. The result, he says, would be to explicitly favor "foreground" processes (assuming those processes are the ones which request lower latency). Distributors could set up defaults for these parameters; users could change them, if they wanted to. 

All of that, Con said, would be a good way to ""move away from the fragile heuristic tweaks and find a longer term robust solution."" The suggestion has not been particularly well received, though. Group scheduling was defended against the "heuristics" label; it is simply an implementation of the scheduling preferences established by the user or system administrator. The session-based component is just a default for how the groups can be composed; it may well be a better default than "no groups," which is what most systems are using now. More to the point, changing that default is easily done. Lennart Poettering's systemd-driven groups are an example; they are managed entirely from user space. Group scheduling is, in fact, quite easy to manage for anybody who wants to set up a different scheme. 

So we'll probably not see Con's knobs added anytime soon - even if somebody does actually create a patch to implement them. What we might see, though, is a variant on that approach where processes could specify exact latency and CPU requirements. A patch for that does exist - it's called the [deadline scheduler](<https://lwn.net/Articles/356576/>). If clever group scheduling turns out not to solve everybody's problem (likely - somebody always has an intractable problem), we might see a new push to get the deadline scheduling patches merged.
