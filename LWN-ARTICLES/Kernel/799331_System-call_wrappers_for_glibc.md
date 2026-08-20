---
title: "System-call wrappers for glibc"
url: https://lwn.net/Articles/799331/
date: "September 19, 2019"
category: System calls
author: "By Jonathan Corbet September 19, 2019 LPC"
---

By **Jonathan Corbet**  
September 19, 2019

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2019-Linux_Plumbers_Conference>)

The GNU C Library has long had a reputation for being hostile to the addition of wrappers for new Linux system calls; that has resulted in many system calls being unsupported by the library for years. That situation is changing, though. During the Toolchain microconference at the 2019 Linux Plumbers Conference, Maciej Rozycki talked about glibc's new attitude toward system-call wrappers, but also served notice that there is still significant work to do for the addition of any new system call. 

Rozycki, who put together the talk with Dmitry Levin, is not the person doing most of this work. He was, instead, "delivering a message from Florian Weimer", who was unable to attend the event. 

For those who might appreciate a bit of background: applications running in user space do not call directly into the kernel; instead, they will call a wrapper function that knows how to invoke the system call of interest. If nothing else, the wrapper will place the system-call arguments in the right locations and do whatever is necessary to invoke a trap into kernel mode. In some cases, the interface implemented by the wrapper can be significantly different from what the kernel provides. 

The provision of a specialized wrapper is not strictly necessary; an application can always gain access to an unsupported system call by way of the [`syscall()`](<http://man7.org/linux/man-pages/man2/syscall.2.html>) function. But, as Rozycki began, there are good reasons to not require applications to do that. There is no type-checking of arguments with `syscall()`, for example. System-call numbers vary from one architecture to the next, even if a Linux kernel is running in both cases, and there can be other ABI differences as well; that makes writing portable [![\[Maciej Rozycki\]](https://static.lwn.net/images/conf/2019/lpc/MaciejRozycki-sm.jpg)](<https://lwn.net/Articles/799336/>) code with `syscall()` difficult. Then, there are the difficulties that come with POSIX threads and thread cancellation in particular. It is just better to have proper C-library support for the system calls that applications need to use. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

So the intention in the glibc project has shifted from blocking system-call wrappers to accepting them. They can't all come in at once, though; each must clear some obstacles first. These include proper documentation in the [glibc manual](<https://www.gnu.org/software/libc/manual/>) and, since glibc is a Free Software Foundation project, copyright-assignment paperwork in place. That last requirement led to a discussion on whether requiring copyright on system-call wrappers amounts to a recognition of ABI copyrights in general — surprisingly, no useful conclusions came from that part of the conversation. The final problem with getting wrappers merged, Rozycki said, is common to all free-software projects: a lack of reviewers to look them over. 

The project has adopted a policy of not emulating system calls in almost all circumstances. If a given system call is not available, the library will return `ENOSYS` and be done with it. System-call emulation has proved to be error-prone, so it will only be done in the most trivial of cases. Glibc also requires that wrapper names be architecture-independent, the alternative being a "maintenance nightmare". If possible, the glibc developers want to add support for all architectures in a single release; otherwise keeping track of things gets difficult. 

Glibc developers have also learned a lesson that has been felt in kernel circles, even if that lesson is still not always taken to heart: multiplexing system calls are painful to support. They make the checking of argument types difficult or impossible, and the situation is even worse for variadic calls (those which take different numbers of arguments for different operations). One result of this aversion to multiplexing system calls may well be that calls like `[futex()](<http://man7.org/linux/man-pages/man2/futex.2.html>)` and `[bpf()](<http://man7.org/linux/man-pages/man2/bpf.2.html>)` will probably be implemented in glibc as a set of independent wrappers, one for each operation. 

There are some specific ABI rules that have been adopted for system-call wrappers. For example, `ssize_t` or `size_t` should be used for all buffer sizes, regardless of the type the kernel uses; that helps to make the purpose of the argument clear. Flags should not have a `long` type, since it's often unclear how the upper 32 bits should be handled on 32-bit machines. Errors should always be returned via `errno`, except for the POSIX threads calls, which are a perennial exception. The glibc developers also feel that their lives would be easier if each new system call had a separate kernel UAPI header file for its associated types and constants. That allows them to include the required information without bringing in any unrelated declarations. 

Rozycki concluded by asking for better cooperation with the C-library projects in general. They should be copied on patches containing ABI changes, for example. I noted that there are often times where C-library developers wish the kernel community had done things differently; how could those be avoided in the future? Members of the audience suggested that more glibc developers should perhaps join the linux-api list. The other suggestion was to "copy Florian on everything". 

Levin added that sometimes documentation can be a limiting issue, ironically it can be worse if a new system call comes out of the kernel community with documentation already written. If the author will not assign copyright for that documentation to the FSF for inclusion in the manual, progress can be blocked; the glibc developers will have to find a way to recreate it without copying the original. This struck some in the audience as a self-imposed problem. 

At the end of session, Christian Brauner said that getting changes into glibc used to be a painful process, but that has changed in recent years. Even the infamous `[gettid()](<http://man7.org/linux/man-pages/man2/gettid.2.html>)` system call, the subject of a [years-long, acrimonious enhancement request](<https://sourceware.org/bugzilla/show_bug.cgi?id=6399>), has now been merged, to the amazement of many. The glibc community is now interacting much more with the kernel community, a change that, hopefully, will continue over time and be echoed on the kernel side. 

[Your editor thanks the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event.]
