---
title: "Ethernet polling and patch-pulling latency"
url: https://lwn.net/Articles/558305/
date: "July 10, 2013"
category: Networking
author: "By Jonathan Corbet July 10, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 10, 2013

Dave Miller's networking git tree is a busy place; it typically feeds over 1,000 changesets into the mainline each development cycle. Linus clearly sees the networking subsystem as being well managed, though, and there are rarely difficulties when Dave puts in his pull requests. So it was surprising to see Linus reject Dave's request for the big 3.11 pull. In the end, it came down to the [low-latency Ethernet device polling patches](<https://lwn.net/Articles/551284/>), which had to go through some urgent repairs while the rest of the networking pull request waited. 

The point of this patch set is to enable low-latency data reception by applications that are willing to busy wait (in the kernel) if data is not available when a `read()` or `poll()` operation is performed on a socket. Busy waiting is normally avoided in the kernel, but, if latency matters more than anything else, some users will be willing to accept the cost of spinning in the kernel if it allows them to avoid the cost of context switches when the data arrives. The hope is that providing this functionality in the kernel will lessen the incentive for certain types of users to install user-space networking stacks. 

Since this patch set was covered here in May, it has seen a few changes. As was predicted, a `setsockopt()` option (`SO_LL`) was added so that the polling behavior could be adjusted on a per-socket basis; previously, all sockets in the system would use busy waiting if the feature was enabled in the kernel. Another flag (`POLL_LL`) was added for the `poll()` system call; once again, it causes busy waiting to happen even if the kernel is otherwise configured not to use it. The runtime kernel configuration itself was split into two sysctl knobs: `low_latency_read` to set the polling time for `read()` operations, and `low_latency_poll` for `poll()` and `select()`. Setting either knob to zero (the default) disables busy waiting for the associated operation. 

When the time came to push the networking changes for 3.11, Dave put the low-latency patches at the top of his list of new features. Linus was [not impressed](<https://lwn.net/Articles/558307/>), though. He had a number of complaints, ranging from naming and documentation through to various implementation issues and the fact that changes had been made to the core `poll()` code without going through the usual channels. He later [retracted](<https://lwn.net/Articles/558308/>) some of his complaints, but still objected to a number of things. For example, he called out code like: 

```
if (ll_flag && can_ll && can_poll_ll(ll_start, ll_time))
```

saying that it ""should have made anybody sane go 'WTF?' and wonder about bad drugs."" More seriously, he strongly disliked the "low-latency" name, saying that it obscured the real effect of the patch. That name, he said, should be changed: 

The "ll" stands for "low latency", but that makes it sound all good. Make it describe what it actually does: "busy loop", and write it out. So that people understand what the actual downsides are. We're not a marketing group. 

So, for example, he was not going to accept `POLL_LL` in the user-space interface; he requested `POLL_BUSY_LOOP` instead. 

Beyond that, Linus disliked how the core polling code worked, saying that it was more complicated than it needed to be. He made a number of suggestions for improving the implementation. Importantly, he wanted to be sure that polling would not happen if the `need_resched` flag is set in the `current` structure. That flag indicates that a higher-priority process is waiting to run on the CPU; when it is set, the current process needs to get out of the way as quickly as possible. Clearly, performing a busy wait for network data would not be the right thing to do in such a situation. Linus did not say that the proposed patch violated that rule, but it was not sufficiently clear to him that things would work as they needed to. 

In response to these perceived shortcomings, Linus refused the entire patch set, putting just over 1,200 changes on hold. He didn't reject the low-latency work altogether, though: 

End result: I think the code is salvageable and people who want this kind of busy-looping can have it. But I really don't want to merge it as-is. I think it was badly done, I think it was badly documented, and I think somebody over-sold the feature by emphasizing the upsides and not the problems. 

As one might imagine, that put a bit of pressure on Eliezer Tamir, the author of the patches in question. The merge window is only two weeks long, so the requested changes needed to be made in a hurry. Eliezer was up to the challenge, though, producing the requested changes in short order. On July 9, Dave posted [a new pull request](<https://lwn.net/Articles/558310/>) with the updated code; Linus pulled the networking tree the same day, though not before posting [a complaint](<https://lwn.net/Articles/558389/>) about some unrelated issues. 

In this case, the last-minute review clearly improved the quality of the implementation; in particular, the user-visible option to `poll()` is now more representative of what it really does (`SO_LL` remains unchanged, but it will become `SO_BUSY_WAIT` before 3.11 is released). The cost, of course, was undoubtedly a fair amount of adrenaline on Eliezer's part as he imagined Dave busy waiting for the fixes. Better review earlier in the process might have allowed some of these issues to be found and fixed in a more relaxed manner. But review bandwidth is, as is the case in most projects, the most severely limited resource of all.
