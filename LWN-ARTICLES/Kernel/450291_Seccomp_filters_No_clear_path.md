---
title: "Seccomp filters: No clear path"
url: https://lwn.net/Articles/450291/
date: "July 7, 2011"
category: "Security-seccomp"
author: "By Jake Edge July 7, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
July 7, 2011

Patches to expand the functionality of seccomp ("secure computing") have been floating around for two years or more without making any real progress into the mainline. There are a number of projects that are interested in using an expanded seccomp, but the patches themselves seem to have run into a "catch-22" situation. There are conflicting visions of how the feature should be added, without a clear sense that any of the options will be acceptable to all of the maintainers involved. That leaves a useful feature without a clear path into the kernel, which is undoubtedly frustrating to some. 

We first [looked at seccomp sandboxing](<https://lwn.net/Articles/332974/>) a little over two years ago, when Adam Langley posted patches that would provide a way for a process to restrict the system calls that it (and its children) could make. The idea is to allow processes to sandbox themselves by choosing which system calls are available, rather than being restricted to just the four hard-coded system calls that the existing seccomp implementation allows (`read()`, `write()`, `exit()`, and `sigreturn()`). The impetus behind Langley's patches was to provide an easier mechanism for sandboxing processes in the Chromium web browser--and to eventually remove the [somewhat convoluted sandbox](<https://lwn.net/Articles/347547/>) that Chromium currently uses on Linux. 

At the time of that proposal, Ingo Molnar [suggested](<https://lwn.net/Articles/332986/>) that Ftrace-style filtering would make the expanded seccomp much more useful. That idea wasn't universally hailed at the time, and the seccomp feature went mostly dormant until it was [restarted](<https://lwn.net/Articles/441232/>) by Will Drewry back in April. Drewry took Molnar's suggestions and implemented a version of seccomp that would allow system calls to be enabled, disabled, or filtered with simple boolean expressions (e.g. `sys_read: (fd == 0)`). 

While Molnar was pleased with the progress, he [didn't think it went far enough](<https://lwn.net/Articles/443099/>) and suggested that a perf-like interface be used instead of `prctl()`, which is used by the existing seccomp. He had some fairly wide-ranging ideas that using perf events in a more active way could lead to better kernel security solutions than the existing Linux Security Modules (LSM) approach provides. Once again, this idea was not universally popular. The LSM developers, in particular, were not enamored by that idea. 

Nevertheless, Drewry [implemented a proof of concept](<http://article.gmane.org/gmane.linux.ports.ppc.embedded/43187>) along the lines of what Molnar had suggested. That led to [complaints](<https://lwn.net/Articles/444745/>) from a somewhat surprising direction, as both Peter Zijlstra and Thomas Gleixner strongly objected to perf being used in an active role. Their responses didn't leave room for any middle ground, with Zijlstra, who is one of the perf maintainers along with Molnar, [saying](<https://lwn.net/Articles/444750/>) that he and Gleixner would NAK ""any and all patches that extend perf/ftrace beyond the passive observing role"". 

All of which led Drewry, who must be feeling a bit whipsawed at this point, to return to the patchset that seemed to have the most support: using Ftrace/perf-style filters, but maintaining the `prctl()` interface that is currently used by seccomp. Linus Torvalds had expressed some [skepticism](<https://lwn.net/Articles/444751/>) that the feature would have any real users, but Drewry [outlined](<http://article.gmane.org/gmane.linux.kernel/1145681>) how it would be used by Chromium, and several other developers spoke up in favor of expanding seccomp, saying that QEMU, Linux containers (LXC), and others would use the feature. Those endorsements, along with resolving some other technical concerns, was enough for Torvalds to [remove his objection](<http://article.gmane.org/gmane.linux.kernel/1146290>) to the feature. But, as might be guessed, Molnar is still not satisfied with the approach. 

When Drewry [reposted](<https://lwn.net/Articles/450389/>) the patchset toward the end of June, and [asked](<https://lwn.net/Articles/450390/>) what the next steps were, Molnar [noted](<https://lwn.net/Articles/450392/>) that his concerns were not being addressed: ""You are pushing the 'filter engine' approach currently, not the (much) more unified 'event filters' approach."" But Drewry is [trying to find a balance](<https://lwn.net/Articles/450393/>) between the needs of the potential users, other maintainers, and Molnar's requests, which is somewhere between difficult and impossible: 

Based on the support from potential API consumers, I believe there is interest in this patch series, and I worry that just like with the last two attempts in the last two years, this series will be relegated to the lwn archives in anticipation of a future solution that uses infrastructure that isn't quite ready. I'm trying to approach a problem that can be addressed today in a flexible, future-friendly way, rather than try to open up a larger cross-kernel impacting patch series that I'm unsure of exactly how to integrate sanely and don't know that I can commit to doing. 

But Molnar is [adamant](<https://lwn.net/Articles/450394/>) that the "filter engine" approach is short-sighted, citing the diffstats of the various implementations as evidence: 

Not doing it right because "it's too much work", especially as the trivial 'proof of concept' prototype already gave us something very promising that worked to a fair degree: 

```
bitmask (2009):  6 files changed,  194 insertions(+), 22 deletions(-)
     filter engine (2010): 18 files changed, 1100 insertions(+), 21 deletions(-)
     event filters (2011):  5 files changed,   82 insertions(+), 16 deletions(-)
```

are pretty hollow arguments to me. That diffstat sums up my argument of proper structure pretty well. 

But, as Drewry [points out](<https://lwn.net/Articles/450412/>), there is still a lot of work to be done to get beyond the proof-of-concept and to a fully fleshed-out solution. Given that the approach has already received several NAKs, doing all of that work has a very uncertain future. Drewry would like to see the feature be available soon, and is concerned that working on the larger problem is likely to delay that significantly, if it can ever get beyond the objections: ""If all the other work is a prerequisite for system call restriction, I'll be very lucky to see anything this calendar year assuming I can even write the patches in that time."" 

Molnar is [undeterred](<https://lwn.net/Articles/450416/>), however, suggesting that there is a path into the kernel through the tree that he co-maintains: 

Do it properly generalized - as shown by the prototype patch. I can give you all help that is needed for that: we can host intermediate stages in -tip and we can push upstream step by step. You won't have to maintain some large in-limbo set of patches. 95% of the work you've identified will be warmly welcome by everyone and will be utilized well beyond sandboxing! That's not a bad starting position to get something controversial upstream: most of the crazy trees are 95% crazy. 

The problem, of course, is that the 5% is the piece that Drewry and others are most interested in seeing (i.e. the system call restrictions for sandboxing) in the kernel. So, what Molnar seems to be offering is a fairly sizable chunk of work that could, in the end, still leave the "interesting" part out in the cold. Molnar may be confident that he can overcome the objections from Zijlstra and Gleixner, but Drewry can hardly be as sanguine. He [describes](<https://lwn.net/Articles/450422/>) the problem as he sees it: 

It seems like a catch-22. There's not a perfectly clear path forward, and anything that looks like the perf-style proof of concept will be NACK'd by other maintainers. While I believe we could lift perf up off its foundation and create a shared location for storing perf events and ftrace events so that they will be inherited the same way (currently nack'd by linus) and walked the same way (kinda), the syscall interface couldn't currently be shared (also nack'd by perf), and creating a new one is possible modeled on the perf one, but it's also unclear what the ABI should be for a generic filtering system. 

Both Zijlstra and Gleixner have been absent from the most recent discussion, so it's a little hard to guess what their thoughts are. In the absence of any kind of posting softening their stances, though, it would be a bad idea to believe that they have changed their minds. 

It's a problem that we have seen before, where a new feature is, to some extent, held hostage to requests that a larger problem be solved. The problem was [discussed](<https://lwn.net/Articles/357805/#rework>) at the 2009 Kernel Summit, where there was agreement that those requests should be advisory in nature, rather than demands. In this case, Molnar is not really demanding that the bigger task be done, just that he is uninterested in taking the code via the -tip tree unless it solves the larger problem. 

It is unclear where things go from here. Drewry said that he would look at trying to do things Molnar's way (""but if my only chance of any form of this being ACK'd is to write it such that it shares code with perf and has a shiny new ABI, then I'll queue up the work for when I can start trying to tackle it""), but it may be a ways off. In the meantime, there are various projects interested in using the feature. 

If falling back to the bitmask version of the feature solves enough of the problem for those projects, there is the possibility of trying to get that into the kernel via another tree (e.g. the security tree). There would undoubtedly be objections from Molnar, but if enough users lined up behind it, that might be a reasonable approach. It would create an ABI that would need to be maintained going forward, which is one of Molnar's objections, but it would solve problems for Chromium and others. 

Steven Rostedt [suggested](<https://lwn.net/Articles/450427/>) adding the seccomp expansion as a discussion item for the Kernel Summit in October, which might provide a path forward. It's likely that most or all of the interested parties will be there (unlike the Linux Security Summit that will be held with Plumbers in September, which was suggested as an alternative). While a face-to-face discussion could be helpful, it might be a stretch to believe that the disagreement between active vs. passive perf could be resolved that way. On the other hand, it could lead to some kind of decree about the proper direction from Torvalds. _That_ could go a long way toward resolving the issue.
