---
title: "TTY-based group scheduling"
url: https://lwn.net/Articles/415740/
date: "November 17, 2010"
category: "Group scheduling; Interactivity; Scheduler-Group scheduling"
author: "By Jonathan Corbet November 17, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 17, 2010

As long as we have desktop systems, there will almost certainly be concerns about desktop interactivity. Many complex schemes for improving interactivity have come and gone over the years; most of them seem to leave at least a subset of users unsatisfied. Miracle cures are hard to come by, but it seems that a recent patch has come close, at least for some users. Interestingly, it is a conceptually simple solution that may not need to be in the kernel at all. 

The core idea behind the completely fair scheduler is its complete fairness: if there are N processes competing for the CPU, each with equal priority, than each will get 1/N of the available CPU time. This policy replaced the rather complicated "interactivity" heuristics found in the O(1) scheduler; it yields better desktop response in most situations. There are places where this approach falls down, though. If a user is running ten instances of the compiler with `make -j 10` along with one video playback application, each process will get a "fair" 9% of the CPU. That 9% may not be enough to provide the video experience that the user was hoping for. So it is not surprising that many users see "fairness" differently; wouldn't be nice if the compilation job as a whole got 50%, while the video application got the other half? 

The kernel has been able to implement that kind of fairness for years though a feature known as [group scheduling](<https://lwn.net/Articles/240474/>). A set of processes placed within a group will each get a fair share of the CPU time allocated to the group as a whole, but groups will, themselves, compete for a fair share of the CPU. So, if the video player were to be placed in one group and the compilation in another, each group would get half of the available processor time. The various processes doing the compilation would then get a fair share of their group's half; they will compete with each other, but not with the video player. This arrangement will ensure that the video player gets enough CPU time to keep up with the stream and any interactivity requirements. 

Groups are thus a nice feature, but they have not seen heavy use since they were merged for the 2.6.24 release. The reasons for that are clear: groups require administrative work and root privileges to set up; most users do not know how to tweak the knobs and would really rather not learn. What has been missing all these years is a way to make group scheduling "just work" for ordinary users. That is the goal of [Mike Galbraith's per-TTY task groups patch](<https://lwn.net/Articles/415742/>). 

In short, this patch automatically creates a group attached to each TTY in the system. All processes with a given TTY as their controlling terminal will be placed in the appropriate group; the group scheduling code can then share time between groups of processes as determined by their controlling terminals. A compilation job is typically started by typing "`make`" in a terminal emulator window; that job will have a different controlling TTY than the video player, which may not have a controlling terminal at all. So the end result is that per-TTY grouping automatically separates tasks run in terminals from those run via the window system. 

This behavior [makes Linus happy](<https://lwn.net/Articles/415748/>); Linus, after all, is just the sort of person who might try to sneak in a quick video while waiting for a highly-parallel kernel compilation. He said: 

So I think this is firmly one of those "real improvement" patches. Good job. Group scheduling goes from "useful for some specific server loads" to "that's a killer feature". 

Others have also reported significant improvements in desktop response, so this feature looks like one which has a better-than-average chance of getting into the mainline in the next merge window. There are, however, a few voices of dissent, most of whom think that the TTY is the wrong marker to use when placing processes in group. 

Most outspoken - as he often is - is Lennart Poettering, who [asserted](<https://lwn.net/Articles/415750/>) that ""Binding something like this to TTYs is just backwards""; he would rather see something which is based on sessions. And, he said, all of this could better be done in user space. Linus was, to put it politely, [unimpressed](<https://lwn.net/Articles/415751/>), but Lennart [came back](<https://lwn.net/Articles/415753/>) with a few lines of bash scripting which achieves the same result as Mike's patch - with no kernel patching required at all. It turns out that working with control groups is not necessarily that hard. 

Linus, however, [still likes the kernel version](<https://lwn.net/Articles/415754/>), mainly because it can be made to "just work" with no user intervention required at all: 

Put another way: if we find a better way to do something, we should _not_ say "well, if users want it, they can do this <technical thing here>". If it really is a better way to do something, we should just do it. Requiring user setup is _not_ a feature. 

In other words, an improvement that just comes with a new kernel is likely to be available to more users than something which requires each user to make a (one-time) manual change. 

Lennart [isn't buying it](<https://lwn.net/Articles/415756/>). A real user-space solution, he says, would not come in the form of a requirement that users edit their `.bashrc` files; it, too, would be in a form that "just works." It should come as little surprise that the form he envisions is systemd; it seems that future plans involve systemd taking over session management, at which time per-session group scheduling will be easy to achieve. He believes that this solution will be more flexible; it will be able to group processes in ways which make more sense for "normal desktop users" than TTY-based grouping. It also will not require a kernel upgrade to take effect. 

Another idea which has been raised is to add a "run in separate group" option to desktop application launchers, giving users an easy way to control how the partitioning is done. 

Linus [seems to be holding his line](<https://lwn.net/Articles/415759/>) on the kernel version of the patch: 

Anyway, I find it depressing that now that this is solved, people come out of the woodwork and say "hey you could do this". Where were you guys a year ago or more? 

Tough. I found out that I can solve it using cgroups, I asked people to comment and help, and I think the kernel approach is wonderful and _way_ simpler than the scripts I've seen. Yes, I'm biased ("kernels are easy - user space maintenance is a big pain"). 

The next merge window is not due until January, though; that is a fair amount of time for people to demonstrate other approaches. If a solution based in user space turns out to be more flexible and effective in the long run, it may yet prevail. That is especially true because merging Mike's patch does not in any way inhibit user-space solutions; if a systemd-based approach shows better results, that may be what the distributors decide to enable. One way or the other, it seems like better interactive response is coming in the near future.
