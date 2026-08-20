---
title: Expedited memory reclaim from killed processes
url: https://lwn.net/Articles/785709/
date: "April 12, 2019"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jonathan Corbet April 12, 2019"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 12, 2019

Running out of memory puts a Linux system into a difficult situation; in the worst cases, there is often no way out other than killing one or more processes to reclaim their memory. This killing may be done by the kernel itself or, on systems like Android, by a user-space out-of-memory (OOM) killer process. Killing a process is almost certain to make somebody unhappy; the kernel should at least try to use that process's memory expeditiously so that, with luck, no other processes must die. That does not always happen, though, in current kernels. [This patch set](<https://lwn.net/ml/linux-kernel/20190411014353.113252-1-surenb@google.com/>) from Suren Baghdasaryan aims to improve the situation, but the solution that results in the end may take a different form. 

The kernel requires processes to clean up their own messes when they exit. So, before a process targeted by an OOM killer can rest in peace, it must first run (in the kernel) to free its memory resources and return them to the system. If the process is running with a low priority, or if it gets hung up waiting for a resource elsewhere in the system, this cleanup work may take a long time. Meanwhile, the system is still struggling, since the hoped-for influx of free memory has not yet happened; at some point, another process will also be killed in the hope of getting better results. 

This problem led to the development of the [OOM reaper](<https://lwn.net/Articles/668126/#reaper>) in 2015. When the kernel's OOM killer targets a process, the OOM reaper will attempt to quickly strip that process of memory that the process will never attempt to access again. But the OOM reaper is not available to user-space OOM killers, which can only send a `SIGKILL` signal and hope that the target exits quickly. The user-space killer never really knows how long it will take for a killed process to exit; as Baghdasaryan pointed out, that means that it must maintain a larger reserve of free pages and kill processes sooner than might be optimal. 

Baghdasaryan's proposal is to add a new flag (`SS_EXPEDITE`) to the (new in 5.1) `pidfd_send_signal()` system call. If that flag is present, the caller has the `CAP_SYS_NICE` capability, and the signal is `SIGKILL`, then the OOM reaper will be unleashed on the killed process. That should result in quicker and more predictable freeing of the target's memory, regardless of anything that might slow down the target itself. 

The comments on the proposal were heavily negative, which is interesting because most of the people involved were supportive of the objective itself. The strongest critic, perhaps, was Michal Hocko (the author of the OOM reaper), who [complained](<https://lwn.net/ml/linux-kernel/20190411105111.GR10383@dhcp22.suse.cz/>) that it ""is abusing an implementation detail of the OOM implementation and makes it an official API"". He questioned whether this capability was useful at all, saying that relying on cleanup speed is ""a fundamental design problem"". Johannes Weiner, though, [argued](<https://lwn.net/ml/linux-kernel/20190411171909.GB5136@cmpxchg.org/>) that the idea was, instead, ""just optimizing the user experience as best as it can"". Others generally agreed that freeing memory quickly when a process is on its way out is a good thing. 

Daniel Colascione [liked the idea](<https://lwn.net/ml/linux-kernel/CAKOZuesgCpyLzs3g=RxyjBMjiMMxDbA2kOZZs3YOqOv=Ri6KgQ@mail.gmail.com/>) but wanted a different interface. Rather than adding semantics to a specific signal, he suggested a new system call along the lines of: 

```
size_t try_reap_dying_process(int pidfd, int flags, size_t max_bytes);
```

This call would attempt to strip memory from the process indicated by `pidfd`, but only if the process is currently dying. The `max_bytes` parameter would tell the kernel how much memory the caller would liked to see freed; that would allow the kernel to avoid doing the full work of stripping the target process if `max_bytes` has been reached. Notably, the reaping would be done in the context of the calling process, allowing user space to determine how important this work is relative to other tasks. 

Matthew Wilcox had [a simpler question](<https://lwn.net/ml/linux-kernel/20190411153313.GE22763@bombadil.infradead.org/>): why not just expedite the reclaim of memory _every_ time a process exits, rather than on specific request? Weiner [agreed](<https://lwn.net/ml/linux-kernel/20190411170540.GA5136@cmpxchg.org/>) with that idea, noting that the work has to be done anyway, and doing it always would avoid the need to add a new interface for that purpose. Daniel Colascione [pointed out](<https://lwn.net/ml/linux-kernel/CAKOZuet8-en+tMYu_QqVCxmkak44T7MnmRgfJBot0+P_A+Qzkw@mail.gmail.com/>), though, that such a mechanism would slow down the delivery of `SIGKILL` in general; that slowdown might be felt when running something like `killall`. 

Baghdasaryan [didn't think](<https://lwn.net/ml/linux-kernel/CAJuCfpGQ8c-OCws-zxZyqKGy1CfZpjxDKMH__qAm5FFXBcnWOw@mail.gmail.com/>) that expedited reaping makes sense all the time. But, he said, there are times when it is critical. ""It would be great if we can identify urgent cases without userspace hints, so I'm open to suggestions that do not involve additional flags"". If such a solution can be found, it is likely to end up as the preferred alternative here. Cleaning up after a killed process is, in the end, the kernel's responsibility, and there is little desire to create a new interface to control how that responsibility is carried out. Solutions that "just work" are thus to be preferred over the addition of yet another Linux-specific API for this purpose.
