---
title: "Short subjects: Realtime, Futexes, and ntfs3"
url: https://lwn.net/Articles/866112/
date: "August 16, 2021"
category: "Filesystems-ntfs3; Futex; Realtime"
author: "By Jonathan Corbet August 16, 2021"
---

By **Jonathan Corbet**  
August 16, 2021

Even in the dog days of (northern-hemisphere) summer, the kernel community is a busy place. There are many developments that show up on your editor's radar, but which, for whatever reason, do not find their way into a full-length feature article. The time has come to catch up with a few of those topics; read on for updates on the realtime patch set, the effort to reinvent futexes, and the ntfs3 filesystem. 

#### Realtime

The realtime preemption story is a long one; it first [showed up on LWN](<https://lwn.net/Articles/106010/>) in 2004. Over the years, this work has had a significant impact on kernel development as a whole; much of what is just seen as part of the core kernel now had its origins in the realtime tree. The code around which the realtime work was initially built — the preemptible locking infrastructure — remains out of the mainline, though. Without the locking changes, the mainline is not able to offer the sort of response-time guarantees that realtime users need. 

The locking infrastructure makes almost all locks, spinlocks included, into sleeping locks; that ensures that a higher-priority task can always take over the processor quickly. It is the sort of change that makes kernel developers nervous, since mistakes in this area can lead to all sorts of subtle problems. For that reason, predicting when the locking code will be merged into the mainline is a fool's game. Your editor knows this well, having confidently [predicted](<https://lwn.net/Articles/263129/>) that it would be merged within a year — in 2007. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Still, one might be tempted to think that the end might be getting closer. Realtime developer Thomas Gleixner has brought the locking infrastructure back to the mailing lists for consideration; [the fifth revision](<https://lwn.net/ml/linux-kernel/20210815203225.710392609@linutronix.de/>) of the 72-part patch set was posted on August 15\. Normally configured kernels should behave about the same with these patches applied, but those configured for realtime operation will have realtime-specific versions of mutexes, wait/wound mutexes, reader/writer semaphores, spinlocks, and reader/writer locks. 

Commentary on this work has slowed; there does not appear to be much in the way of objections at this point — though it must be noted that Linus Torvalds has not yet made his feelings known on the subject. Unless something surprising comes up, it might just be that the core realtime code will finally find its way into the mainline. Your editor, however, is too old, wise, and cowardly to venture a guess as to when that will happen. 

#### A smaller step for futex2

Perhaps the number of comments on the realtime changes is low because most developers fear the prospect of digging into code of that complexity. There are, however, places in the kernel that are even more frightening; the futex subsystem is surely one of them. Futexes provide fast mutexes for user space; they started out as a simple subsystem but failed to remain that way. Over time, it has become clear that futexes could do with a number of improvements to make them better suited for current workloads and, at the same time, to move beyond the multiplexer [`futex()`](<https://man7.org/linux/man-pages/man2/futex.2.html>) system call. 

For some time now, André Almeida has been pushing in that direction with the [futex2 proposal](<https://lwn.net/Articles/823513/>). This work would split the futex functionality into several single-purpose system calls, support multiple lock sizes, and more. While there has been interest in this work, progress has been slow (to put it charitably); it seems as if the kernel is no closer to a new futex subsystem than it was a year or two ago. 

In an attempt to push this project forward, Almeida has posted [a new patch set](<https://lwn.net/ml/linux-kernel/20210805190405.59110-1-andrealmeid@collabora.com/>) with significantly reduced ambitions. Rather than introduce a whole new subsystem with its own system calls, this series adds exactly one system call that works with existing futexes: 

```
struct futex_waitv {
            uint64_t val;
            void *uaddr;
            unsigned int flags;
        };
    
        int futex_waitv(struct futex_waitv *waiters, unsigned int nr_futexes,
                        unsigned int flags, struct timespec *timo);
```

This function will cause the calling process to wait on several futexes simultaneously, returning when one or more of them can be acquired (or the timeout expires). That functionality is not supported by the current futex API, but it turns out to be especially useful for game engines, which perform significantly better when using the new system call. [This documentation patch](<https://lwn.net/ml/linux-kernel/20210805190405.59110-5-andrealmeid@collabora.com/>) describes the new API in more detail. 

This patch set has drawn no comments in the week since it was posted. Assuming that silence implies a lack of objections rather than a lack of interest, this piece of the futex2 work might make it into a mainline release before too long. Whether the rest of the futex2 work will follow depends on how strong the use cases driving it are; if `futex_waitv()` solves the worst problems, there might not be much motivation to push the other changes. 

#### Waiting for ntfs3

The kernel has long had an implementation of the NTFS filesystem, but it has always suffered from performance and functionality problems; the user community would gladly trade it for something better. By all accounts, the ntfs3 implementation posted by Konstantin Komarov is indeed something better, but it is still not clear when it will be merged; this work was first [posted](<https://lwn.net/Articles/828824/>) one year ago, and [version 27](<https://lwn.net/ml/linux-kernel/20210729134943.778917-1-almaz.alexandrovich@paragon-software.com/>) of the patch set was posted on July 29\. 

The delay in accepting this work is proving frustrating to users; [this complaint from Neal Gompa](<https://lwn.net/ml/linux-kernel/20210715215029.2689112-1-ngompa13@gmail.com/>) is typical: 

> I know that compared to all you awesome folks, I'm just a lowly user, but it's been frustrating to see nothing happen for months with something that has a seriously high impact for a lot of people. 
> 
> It's a shame, because the ntfs3 driver is miles better than the current ntfs one, and is a solid replacement for the unmaintained ntfs-3g FUSE implementation. 

Torvalds has [said](<https://lwn.net/ml/linux-kernel/CAHk-=whfeq9gyPWK3yao6cCj7LKeU3vQEDGJ3rKDdcaPNVMQzQ@mail.gmail.com/>) that maybe it is time to merge this code, but that still may not happen right away. 

The biggest holdup for ntfs3 at the moment would appear to be concerns about the level of development effort behind it. From the public evidence, it seems that ntfs3 is a one-person project, and that makes other filesystem developers nervous. Those developers have been reporting test failures for ntfs3 that have gone unfixed. Meanwhile, Komarov is sometimes unresponsive to questions; various comments on the [version 26 posting](<https://lwn.net/ml/linux-kernel/20210402155347.64594-1-almaz.alexandrovich@paragon-software.com/>) (from early April) got no answers, for example. This sort of silence gives the impression that ntfs3 does not have a lot of effort behind it. (It's worth noting that some other developers [have been happy](<https://lwn.net/ml/linux-kernel/YO8LOKR/vRUgggTx@casper.infradead.org/>) with the level of response from Komarov). 

Unsurprisingly, the filesystem developers are unenthusiastic about the prospect of taking on a new NTFS implementation that may turn out to have serious problems and which does not come with a promise of reliable support. For ntfs3 to be merged, those fears will need to be addressed somehow. One way for that to happen, as [suggested](<https://lwn.net/ml/linux-kernel/YQrAsGBmVeKQp+Z9@mit.edu/>) by Ted Ts'o, would be for other developers, perhaps representing one or more distributors that would like to see a better NTFS implementation in the kernel, to start contributing patches to ntfs3 and commit to helping with its maintenance going forward.
