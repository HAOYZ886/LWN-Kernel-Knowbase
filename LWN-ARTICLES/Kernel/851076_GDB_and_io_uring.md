---
title: GDB and io_uring
url: https://lwn.net/Articles/851076/
date: "March 31, 2021"
category: io uring
author: "By Jake Edge March 31, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
March 31, 2021

A problem reported when attaching GDB to programs that use [io_uring](<https://lwn.net/Articles/810414/>) has led to a flurry of potential solutions, and one that was merged into Linux 5.12-rc5. The problem stemmed from a change made in the [5.12 merge window](<https://lwn.net/Articles/847451/>) to how the threads used by io_uring were created, such that they became associated with the process using io_uring. Those "I/O threads" were treated specially in the kernel, but that led to the problem with GDB (and likely other [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>)-using programs). The solution is to treat them like other threads because it turned out that trying to make them special caused more problems than it solved. 

Stefan Metzmacher [reported](<https://lwn.net/ml/io-uring/d1a8958c-aec7-4f94-45f8-f4c2f2ecacff@samba.org/>) the problem to the io-uring mailing list on March 20\. He tried to attach GDB to the process of a program using io_uring, but the debugger went ""into an endless loop because it can't attach to the io_threads"". `PF_IO_WORKER` threads are used by io_uring for operations that might block; he followed up the bug report with two patch sets that would hide these threads in various ways. The idea behind hiding them is that if GDB cannot see the threads, it will not attempt to attach to them. Prior to 5.12, the threads existed but were not associated with the io_uring-using process, so GDB would not see them. 

It is, of course, less than desirable for developers to be unable to run a debugger on code that uses io_uring, especially since io_uring support in their application is likely to be relatively new, thus it may need more in the way of debugging. The maintainer of the io_uring subsystem, Jens Axboe, quickly stepped in to help Metzmacher solve the problem. Axboe [posted](<https://lwn.net/ml/linux-kernel/20210320153832.1033687-1-axboe%40kernel.dk/>) a patch set that included a way to hide the `PF_IO_WORKER` threads, along with some tweaks to the signal handling for these threads; in particular, he removed the ability for them to receive signals at all. 

That made Eric W. Biederman somewhat uncomfortable; he [asked](<https://lwn.net/ml/linux-kernel/m1eeg9bxyi.fsf@fess.ebiederm.org/>) why the io_uring threads could not take signals, and [`SIGSTOP` in particular](<https://lwn.net/ml/linux-kernel/m1sg4paj8h.fsf@fess.ebiederm.org/>). In order to attach to a running process, `ptrace()` uses `SIGSTOP`, but the I/O threads lack much of the normal user-space context that would allow handling signals. Linus Torvalds [explained](<https://lwn.net/ml/linux-kernel/CAHk-=wjLMy+J20ZSBec4iarw2NeSu5sWXm6wdMH59n-e0Qe06g@mail.gmail.com/>) that signal handling is done when a thread returns to user space, but that does not happen for kernel threads; he [described](<https://lwn.net/ml/linux-kernel/CAHk-=whyL6prwWR0GdgxLZm_w-QWwo7jPw_DkEGYFbMeCdo8YQ@mail.gmail.com/>) that further in another message: 

> SIGSTOP handling is fundamentally done at signal handling time, and signal handling is fundamentally done at "return to user space" time. 
> 
> End result: you cannot send kernel threads any signals at all, unless it _explicitly_ handles them manually. SIGSTOP isn't different from user space delivery of an "actual" signal handler in this respect. 
> 
> And practically speaking, the only signal a kernel thread generally can handle is SIGKILL (and exit on it). 
> 
> [...] I really think IO threads need to not participate, because they simply cannot handle signals in any sane manner. 

A few days later, Axboe [posted](<https://lwn.net/ml/linux-kernel/20210325164343.807498-1-axboe@kernel.dk/>) another version of the patch set with a longer description of the problem and the proposed solution: 

> Stefan reports that attaching to a task with io_uring will leave gdb very confused and just repeatedly attempting to attach to the IO threads, even though it receives an -EPERM every time. This patchset proposes to skip PF_IO_WORKER threads as same_thread_group(), except for accounting purposes which we still desire. 
> 
> We also skip listing the IO threads in /proc/<pid>/task/ so that gdb doesn't think it should stop and attach to them. This makes us consistent with earlier kernels, where these async threads were not related to the ring owning task, and hence gdb (and others) ignored them anyway. 

But it seems those patches went a bit too far; Biederman [pointed out](<https://lwn.net/ml/linux-kernel/m1ft0j3u5k.fsf@fess.ebiederm.org/>) that the threads would no longer show up in `/proc` at all, which would hide them from `top` and other diagnostic tools. Torvalds [noted](<https://lwn.net/ml/linux-kernel/CAHk-=wjOXiEAjGLbn2mWRsxqpAYUPcwCj2x5WgEAh=gj+o0t4Q@mail.gmail.com/>) that it also hides them from `ps`, which makes him ""think that hiding them is the wrong model"". There was some [discussion](<https://lwn.net/ml/linux-kernel/CAHk-=wg1XpX=iAv=1HCUReMbEgeN5UogZ4_tbi+ehaHZG6d==g@mail.gmail.com/>) of putting the threads under a different name in the `/proc` hierarchy, but Axboe [thought](<https://lwn.net/ml/linux-kernel/3a1c02a5-db6d-e3e1-6ff5-69dd7cd61258@kernel.dk/>) it might work for some utilities but would likely ""mess up _something_"". Biederman [said](<https://lwn.net/ml/linux-kernel/m1zgyr2ddh.fsf@fess.ebiederm.org/>) that there needed to be some mechanism to tell GDB (and other debuggers) that these threads are special: ""I suspect getting -EPERM (or possibly a different error code) when attempting attach is the right [way] to know that a thread is not available to be debugged."" 

Axboe mentioned in the patch cover letter that the underlying problem might really be a GDB bug and Biederman seemed to agree, but Oleg Nesterov [took exception](<https://lwn.net/ml/linux-kernel/20210325204014.GD28349@redhat.com/>) to that: ""The kernel changed the rules, and this broke gdb."" But Biederman [argued](<https://lwn.net/ml/linux-kernel/m135wj0xj8.fsf@fess.ebiederm.org/>) that it was not strictly a regression; ""It is gdb not handling new functionality."" Though, even if it was a GDB bug, Axboe [said](<https://lwn.net/ml/linux-kernel/27a54fc2-865a-d90b-2429-d91170be5873@kernel.dk/>), it simply takes too long for updates to GDB to make their way to users to fix things that way; beyond that, ""I doubt that gdb is the only thing that'd fall over, not expecting threads in there that it cannot attach to."" 

So, some kind of solution where everything "just works" was desired—and that is seemingly exactly what Torvalds [came up with](<https://lwn.net/ml/linux-kernel/CAHk-=wgUcVeaKhtBgJO3TfE69miJq-krtL8r_Wf_=LBTJw6WSg@mail.gmail.com/>): 

> Actually, maybe the right model is to simply make all the io threads take signals, and get rid of all the special cases. 
> 
> Sure, the signals will never be delivered to user space, but if we 
> 
>   * just made the thread loop do "get_signal()" when there are pending signals 
>   * allowed ptrace_attach on them 

> 
> they'd look pretty much like regular threads that just never do the user-space part of signal handling. 
> 
> The whole "signals are very special for IO threads" thing has caused so many problems, that maybe the solution is simply to _not_ make them special? 

Axboe [agreed](<https://lwn.net/ml/linux-kernel/ad21da2b-01ea-e77c-70b2-0401059e322b@kernel.dk/>) that it made more sense to ""just embrace the signals, and have everything just work by default"". To that end, he [tried](<https://lwn.net/ml/linux-kernel/f9bc0bac-2ad9-827e-7360-099e1e310df5@kernel.dk/>) attaching GDB after making changes along those lines and was successful. That led to [version 1](<https://lwn.net/ml/linux-kernel/20210326003928.978750-1-axboe@kernel.dk/>) of the ""allow signals for IO threads"" patch set on March 25 and, after a few fixes, [version 2](<https://lwn.net/ml/linux-kernel/20210326155128.1057078-1-axboe@kernel.dk/>) on March 26\. The latter was promptly [picked up for 5.12-rc5](<https://lwn.net/Articles/850986/>) on March 28; it also reverts some of the earlier attempts at fixes that had been picked up for 5.12-rc4 a week earlier. 

The idea is that the I/O threads simply take signals like other threads and processes rather than be a special case, which cleans things up substantially. As Axboe put it in the cover letter of the first version, it is plain to see in hindsight: 

> As with most other good ideas, it's obvious once you hear it. The fact that we end up with _zero_ special cases with this is a clear sign that this is the right way to do it indeed. The fact that this series is 2/3rds revert further drives that point home. 

In the end, it would seem that a much better solution came about; at least in part by Torvalds stepping back and reconsidering some of the assumptions. While `PF_IO_WORKER` threads cannot really do anything with signals that get sent to them, there is no real need to reject them either. Once that was recognized, the patches were fairly straightforward it seems. And, meanwhile, an unpleasant situation for developers of io_uring-using code was avoided.
