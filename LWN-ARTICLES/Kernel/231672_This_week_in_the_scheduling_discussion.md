---
title: This week in the scheduling discussion
url: https://lwn.net/Articles/231672/
date: "April 24, 2007"
category: "Interactivity; Scheduler-Completely fair scheduler"
author: 
---

In [last week's scheduler timeslice](<http://lwn.net/Articles/230574/>), Ingo Molnar had introduced his "completely fair scheduler" patch and Staircase Deadline scheduler author Con Kolivas had retreated in a bit of a sulk. Since then, Con has returned and posted several new revisions of the SD scheduler, but with little discussion. His intent, seemingly, is to raise the bar and ensure that whatever scheduler does eventually replace the current system is the best possible - a goal which few should be able to disagree with. 

Most of the discussion, though, has centered around the CFS scheduler. Several testers have reported good results, but others have noted some behavioral regressions. These problems, like most of the others over the years, involve the X Window System. So potential solutions are being discussed yet again. 

The classic response to X interactivity problems is to renice the X server. But this solution seems like a bit of a hack to many, so scheduler work has often been aimed at eliminating the need to run X at a higher priority. Con Kolivas [questions this goal](<https://lwn.net/Articles/231677/>): 

The one fly in the ointment for linux remains X. I am still, to this moment, completely and utterly stunned at why everyone is trying to find increasingly complex unique ways to manage X when all it needs is more cpu. Now most of these are actually very good ideas about _extra_ features that would be desirable in the long run for linux, but given the ludicrous simplicity of renicing X I cannot fathom why people keep promoting these alternatives. 

Avoiding renicing remains a goal of CFS, but it's interesting to see that the v4 CFS patch does renice X - automatically. More specifically, the scheduler bumps the priority level of any process performing hardware I/O (as seen by calls to `ioperm()` or `iopl()`, the loop block device thread, and worker threads associated with workqueues. With the X server automatically boosted (as a result of its `iopl()` use), it does tend to be more responsive. 

While giving kernel threads a priority boost might make sense in the long term, Ingo sees renicing X as a temporary hack. The real solution to the problem seems to involve two different approaches: CPU credit transfers between processes and group scheduling. 

Remember that, with the CFS scheduler, each process accumulates a certain amount of CPU time which is "owed" to it; this time is earned by waiting while others use the processor. This mechanism can enforce basic fairness between processes, in that each one gets something very close to an equal share of the available CPU time. Whether this calculation is truly "fair" depends on how one judges fairness; if the X server is seen as performing work for other processes, then fairness would call for X to share in the credit accumulated by those other processes. Linus [has been pushing](<https://lwn.net/Articles/231682/>) for a solution along these lines: 

The "perfect" situation would be that when somebody goes to sleep, any extra points it had could be given to whoever it woke up last. Note that for something like X, it means that the points are 100% ephemeral: it gets points when a client sends it a request, but it would *lose* the points again when it sends the reply! 

The [CFS v5 patch](<http://lwn.net/Articles/231476/>) has the beginnings of support for this mode of operation. Automatic transfers of credit are not there, but there is a new system call: 

```
long sched_yield_to(pid_t pid);
```

This call gives up the processor much like `sched_yield()`, but it also gives half of the yielding process's credit (if it has any) to the process identified by `pid`. This system call could be used by (for example) the X libraries as a way to explicitly transfer credit to the X server. There is currently no way for the X server to give back the credit it didn't use; Ingo has [mentioned](<https://lwn.net/Articles/231684/>) the notion of a `sched_pay()` system call for that purpose. There's also no way to ensure that X uses the credit for work done on the yielding process's behalf; it could just as easily squander it on wobbly window effects. But it's a step in the right direction. 

A further step, in a highly prototypical form, is Ingo's [scheduler economy patch](<http://lwn.net/Articles/231790/>). This mechanism allows kernel code to set up a scheduler "work account"; processes can then make deposits to and withdrawls from the account with: 

```
void sched_pay(struct sched_account *account);
        void sched_withdraw(struct sched_account *account);
```

At this point, deposits and withdrawls all involve a fixed amount of CPU time. The Unix-domain socket code has been modified to create one of these accounts associated with each socket. Any non-root process (X clients, for example) writing to a socket will also make a deposit into the work account; root-owned processes (the X server, in particular) reading messages also withdraw from the account. It's all a proof of concept; a real implementation would require a rather more sophisticated API. But the proof does show that X clients can convey some of their CPU credits to the server when processor time is scarce. 

The other idea in circulation is per-user or group scheduling. Here, the idea is to fairly split CPU time between users instead of between processes. If one user is running a single text editor process when another starts a kernel build with `make -j 100`, the scheduler will have 101 processes all contending for the CPU. The current crop of fair schedulers will divide the processor evenly between all of them, allowing the kernel build to take over while the text editor must make do with less than 1% of the available CPU time. This situation may be just fine with kernel developers, but one can easily argue that the right split here would be to confine the kernel build to half of the available time while allowing the text editor to use the other half. 

That is the essence of per-user scheduling. Among other things, it could ease the X interactivity problem: since X runs as a different user (root, normally), it will naturally end up in a separate scheduling group with its own fair share of the processor. Linus has been pushing hard for group scheduling as well (see the [quote of last week](<http://lwn.net/Articles/230983/>)). Ingo [responds](<https://lwn.net/Articles/231685/>) that group scheduling is on his mind - he just hasn't gotten around to it yet: 

Firstly, i have not neglected the group scheduling related CFS regressions at all, mainly because there _is_ already a quick hack to check whether group scheduling would solve these regressions: renice. And it was tried in both of the two CFS regression cases i'm aware of: Mike's X starvation problem and Willy's "kevents starvation with thousands of scheddos tasks running" problem. And in both cases, applying the renice hack [which should be properly and automatically implemented as uid group scheduling] fixed the regression for them! So i was not worried at all, group scheduling _provably solves_ these CFS regressions. I rather concentrated on the CFS regressions that were much less clear. 

In other words, the automatic renicing described above is not a permanent solution; instead, it's more of a proof of concept for group scheduling. Ingo goes on to say that there's a lot of other important factors in getting interactive scheduling right; in particular, nanosecond accounting and strict division of CPU time were needed. Once all of those details are right, one can start thinking about the group scheduling problem. 

So there would appear to be some work yet to be done on the CFS scheduler. That will doubtless happen; meanwhile, however, Linus has [complained](<https://lwn.net/Articles/231686/>) that some of this effort may be misdirected at the moment: 

Anyway, I'd ask people to look a bit at the current *regressions* instead of spending all their time on something that won't even be merged before 2.6.21 is released, and we thus have some more pressing issues. Please? 

One might argue that any work which is intended for the upcoming 2.6.22 merge window needs to be pulled into shape now. But the replacement of the CPU scheduler is likely to take a little bit longer than that. Given the number of open questions - and the amount of confidence replacing the scheduler requires - any sort of movement for 2.6.22 seems unlikely.
