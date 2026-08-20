---
title: "The edge-triggered misunderstanding"
url: https://lwn.net/Articles/864947/
date: "August 5, 2021"
category: Android; Epoll
author: "By Jonathan Corbet August 5, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 5, 2021

The [Android 12 beta release](<https://blog.google/products/android/android-12-beta/>) first appeared in May of this year. As is almost obligatory, this release features ""the biggest design change in Android's history""; what's an Android release without requiring users to relearn everything? That historical event was not meant to include one change that many beta testers are noticing, though: a kernel regression that breaks a significant number of apps. This problem has just been fixed, but it makes a good example of why preventing regressions can be so hard and how the kernel project responds to them when they do happen. 

Back in late 2019, David Howells [made some changes to the pipe code](<https://lwn.net/ml/linux-kernel/157558502272.10278.8718685637610645781.stgit@warthog.procyon.org.uk/>) to address a couple of problems. Unfortunately, that work caused the sort of regression that the kernel community finds most unacceptable: it [slowed down (or even hung) kernel builds](<https://lwn.net/ml/linux-kernel/CAHk-=wj9pprDUeoJd5EeN-2x7+GXdSsm44mSv1y9f5e7MrTZ2A@mail.gmail.com/>). After [an extensive discussion](<https://lwn.net/ml/linux-kernel/20191206135604.GB2734@twin.jikos.cz/>), an unfortunate interaction with the GNU `make` job server was identified and a [a fix by Linus Torvalds was applied](<https://git.kernel.org/linus/1b6b26ae7053>) that made the problem go away. The 5.5 kernel was released shortly afterward, kernel builds sped back up, and the problem was deemed to have been solved. 

#### Not done yet

At the end of July, Sandeep Patil [informed the community](<https://lwn.net/ml/linux-kernel/20210729222635.2937453-1-sspatil@android.com/>) that, while the GNU `make` problem may have been fixed, the fix created a new problem of its own. He included a patch to revert Torvalds's fix. That revert clearly was never going to be applied on its own — kernel developers still lack an appetite for slower builds — but it did spark an investigation of the real problem. 

The 2019 pipe rework and subsequent fix had been much more painful than Torvalds thought they should be, so he made a number of changes to both the organization and the behavior of the code. Specifically, an important change was made to how pipes work with system calls like [`epoll_wait()`](<https://man7.org/linux/man-pages/man2/epoll_wait.2.html>), [`poll()`](<https://man7.org/linux/man-pages/man2/poll.2.html>), and [`select()`](<https://man7.org/linux/man-pages/man2/select.2.html>). If the desired type of I/O is not possible without blocking, these calls will put the calling process onto a wait queue. When the situation changes (data can now be read or written), the processes on the appropriate wait queue are woken so that they can request their I/O. 

The 2019 fix changed the way that wakeup was performed. Before, a write to a pipe would unconditionally wake any waiting readers; indeed, it could wake them multiple times in a single system call. The fix changed this behavior to only perform the wakeup if the pipe buffer was empty at the start of the operation; a write to a pipe that already contained data waiting to be read would simply add the new data without causing the wakeup. The reasoning behind this change was straightforward enough: the polling system calls will return immediately if data is already available for reading, so there should not _be_ any waiters if the pipe has available data. 

#### On the edge

There is, however, a mode to `epoll_wait()` called "edge triggered" (or `EPOLLET`) that behaves a little differently. A process requesting an edge-triggered wait will _not_ see an immediate return from `epoll_wait()` if there is available data; instead, it will wait until the situation changes. At least, that is how it worked before the 2019 patches. Once the pipe driver stopped performing wakeups any time that data arrived, processes initiating edge-triggered waits when there was already data available would not see the "edge" and would not wake. 

There was, evidently, reason to wonder if this kind of problem would ever arise. The previous version of `pipe_write()` included this helpful comment: 

```
/* Always wake up, even if the copy fails. Otherwise
         * we lock up (O_NONBLOCK-)readers that sleep due to
         * syscall merging.
         * FIXME! Is this really true?
         */
```

It turns out that it was really true. There are a number of Android libraries, such as [Realm](<https://github.com/realm/realm-core>), that depend on getting edge-triggered wakeups even if there had been data waiting in the pipe before the `epoll_wait()` call was made. The purpose, evidently, is to wait until the pipe buffer is full, then read all of the data in a single operation. Those libraries broke when the 5.10 kernel went into the Android 12 beta, bringing down a set of apps with them. Realm has since worked around the problem but, as Patil pointed out, the joy of bundling means that ""it will be a while before all applications incorporate the updated library"". Fixing the kernel would repair the problem for all apps. 

There seems to be widespread agreement that these libraries manifest a misunderstanding of what "edge-triggered" means and are using the edge-triggered mode incorrectly. As Torvalds [explained](<https://lwn.net/ml/linux-kernel/CAHk-=witY33b-vqqp=ApqyoFDpx9p+n4PwG9N-TvF8bq7-tsHw@mail.gmail.com/>): 

> This is literally an epoll() confusion about what an "edge" is. 
> 
> An edge is not "somebody wrote more data". An edge is "there was no data, now there is data". 
> 
> And a level triggered event is *also* not "somebody wrote more data". A level-triggered signal is simply "there is data". 
> 
> Notice how neither edge nor level are about "more data". One is about the edge of "no data" -> "some data", and the other is just a "data is available". 

That, however, is not how edge-triggered operation was implemented for pipes. Unsurprisingly, in a demonstration of [Hyrum's law](<https://www.hyrumslaw.com/>) in action, applications began to rely on the actual behavior of the system rather than the intended semantics. The [`epoll()`](<https://man7.org/linux/man-pages/man7/epoll.7.html>) man page agrees with Torvalds's description, describing just the sort of blocking behavior experienced by the broken apps. In the distant past, kernel developers might have just answered that these libraries are doing it wrong. But that's not how the kernel works now; thus, Torvalds continued: 

> But we have the policy that regressions aren't about documentation or even sane behavior. 
> 
> Regressions are about whether a user application broke in a noticeable way. 

That interpretation of "regression" requires that the problem be fixed. And indeed that was done with [a new fix](<https://git.kernel.org/linus/3a34b13a88ca>) that was merged for 5.14-rc4 at the end of July and was included in the 5.10.56 and 5.13.8 stable updates. This patch does not quite restore the old behavior; specifically, it will only perform one wakeup per write operation. It does appear to have fixed the problem, though. 

#### Problem solved?

The 5.5 kernel was released in January 2020 — a time when few of us understood what was about to descend upon us, and a significant kernel regression is just another addition to the list. This regression thus endured for a year and a half, and found its way into the 5.10 long-term-stable release last December. The fact that it only surfaced now suggests a testing gap among certain users; happily, it was caught before the next Android release was finalized. 

The ominous question that remains, though, is whether, in that year and a half, any applications became dependent on the newer semantics. And, indeed, there is already [a report](<https://lwn.net/ml/linux-kernel/20210802024945.GA8372@xsang-OptiPlex-9020/>) (from Intel's automated test system) that the [hackbench](<https://manpages.ubuntu.com/manpages/bionic/man8/hackbench.8.html>) benchmark regressed by nearly 13% after the latest fix was applied. Torvalds [responded](<https://lwn.net/ml/linux-kernel/CAHk-=whr-zgf41B0BPsWnL-WWqhdk71rKffhV0fzdCuem3=SCQ@mail.gmail.com/>) that he is ""not sure how much hackbench really matters"" and that the regression ""probably doesn't matter one whit"". Even so, he posted [a new patch](<https://lwn.net/ml/linux-kernel/CAHk-=whYu7f=itjJJTfAWdDN2Baz0whdU1bzxh_5RAfj+LyB3g@mail.gmail.com/>) that provides something closer to the older behavior — but only if the pipe is being used with one of the polling functions. Should it turn out that the hackbench regression does matter, there will be a fix in hand. 

If the latest fix breaks something else, though, kernel developers may have a difficult choice to make. It is possible that there is no way forward without leaving a broken application somewhere; this is why it is important to catch regressions early. With luck, that sort of breakage won't happen and this particular episode will now be finally done.
