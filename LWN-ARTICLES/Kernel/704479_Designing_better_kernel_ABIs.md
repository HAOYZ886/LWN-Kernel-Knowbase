---
title: Designing better kernel ABIs
url: https://lwn.net/Articles/704479/
date: "October 26, 2016"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet October 26, 2016 Kernel
Recipes"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 26, 2016

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2016-Kernel_Recipes>)

Michael Kerrisk started his [2016 Kernel Recipes](<https://kernel-recipes.org/en/2016/>) talk by noting that the man pages collection, which he maintains, now documents over 2,200 interfaces at the kernel level and above; it adds up to about 2,500 printed pages. In the course of writing and collecting all this documentation, he has seen a lot of ABIs. There are many lessons to be learned from this experience, he said, if we want to create better ABIs in the future. 

There is a whole set of obvious goals that one might want to meet when designing an ABI. It should, naturally, be bug-free and as simple as possible, while being extensible and maintainable. Compliance with existing standards is important as well. Presumably, he said, we all agree on these goals — but we repeatedly fail on all of them. 

#### A history of failure

Failure comes in many forms. One of those, of course, is simple bugs. "Show me a new interface," Kerrisk said, "and I'll show you a bug." There is insufficient pre-release testing of new ABIs, meaning that too many bugs slip through. That, in turn, leads to pain for user space, which may need to carry special-case code to handle the bugs found in different kernel versions. 

Interface inconsistencies find their way in at a number of levels, including at the design level; he pointed out that the kernel has about a half-dozen different architecture-dependent [`clone()`](<http://man7.org/linux/man-pages/man2/clone.2.html>) interfaces. There are also behavioral inconsistencies that can create ongoing pain for application developers. Consider, for example, the [`mlock()`](<http://man7.org/linux/man-pages/man2/mlock.2.html>) system call: 

```
int mlock(const void *addr, size_t len);
```

When `mlock()` is called, the `addr` argument will be rounded down to the nearest page boundary, while the end of the range (`addr`+`len`) will be rounded up. Thus, calling `mlock(4000, 6000)` will affect memory addresses zero through 12287. Now consider [`remap_file_pages()`](<http://man7.org/linux/man-pages/man2/remap_file_pages.2.html>): 

```
int remap_file_pages(void *addr, size_t len, int prot,
                             size_t pgoff, int flags);
```

This system call rounds `len` _downward_ , so a call to: 

```
remap_file_pages(4000, 6000, ...);
```

will affect the range from zero to 4095. That sort of surprising behavioral difference, he said, is an ongoing source of bugs. Another breeding ground is the set of system calls that change the attributes of another process; these include [`setpriority()`](<http://man7.org/linux/man-pages/man2/getpriority.2.html>), [`ioprio_set()`](<http://man7.org/linux/man-pages/man2/ioprio_set.2.html>), [`migrate_pages()`](<http://man7.org/linux/man-pages/man2/migrate_pages.2.html>), and more. Each of these must check the credentials of the caller and decide when an unprivileged caller will be able to carry out the requested action; each interface has a different set of rules. 

What about maintainability? System calls [should normally include a `flags` argument](<https://lwn.net/Articles/585415/>), but that has often not been the case. As a result, the number of system calls seems to grow without bound. [`umount()`](<http://man7.org/linux/man-pages/man2/umount2.2.html>) lacked a `flags` argument, so now we have `umount2()`. For similar reasons, the kernel offers [`preadv2()`](<http://man7.org/linux/man-pages/man2/readv.2.html>), [`epoll_create1()`](<http://man7.org/linux/man-pages/man2/epoll_create.2.html>), [`renameat2()`](<http://man7.org/linux/man-pages/man2/rename.2.html>), and more. In some cases, the original interface was a historical legacy, but others "we did to ourselves." A related problem is [a failure to check for unknown flags](<https://lwn.net/Articles/588444/>), as seen in [`sigaction()`](<http://man7.org/linux/man-pages/man2/rt_sigaction.2.html>), [`recv()`](<http://man7.org/linux/man-pages/man2/recv.2.html>), [`clock_nanosleep()`](<http://man7.org/linux/man-pages/man2/clock_nanosleep.2.html>), [`msgrcv()`](<http://man7.org/linux/man-pages/man2/msgrcv.2.html>), [`semget()`](<http://man7.org/linux/man-pages/man2/semget.2.html>), [`semop()`](<http://man7.org/linux/man-pages/man2/semtimedop.2.html>), [`open()`](<http://man7.org/linux/man-pages/man2/open.2.html>), and more. Among other things, failure to return an error on an unknown flag means that user space cannot check whether a given feature is supported or not. 

In short, he said, decentralized design as seen in the kernel community has its advantages, but it fails badly if the goal is a _coherent_ design. As an example, consider [capabilities](<http://man7.org/linux/man-pages/man7/capabilities.7.html>). When a new privileged feature is added to the kernel, the question arises: should a new capability be added to control access to it? Nobody wants to see an explosion in the number of capabilities, so it is generally deemed preferable to use an existing one. In practice, the existing one is almost invariably `CAP_SYS_ADMIN`, which is tested for in some 40% of all cases. It is, he said, "[the new root](<https://lwn.net/Articles/486306/>)", and the goal of finer-grained privilege checking has not been achieved. The first version of the control-group ABI was also plagued by inconsistencies. 

In the end, he said, "we are just traditionalists" following and upholding a long history of Unix ABI mess-ups. The problem, of course, is that interface design is hard, and errors normally cannot be fixed without breaking existing user-space programs. So thousands of programs have to live with the consequences of our ABI mistakes for decades. We really need to get better at getting things right the first time. 

#### Avoiding failure

How do we do that? A lot of it comes down to review and testing. Unlike some other parts of the kernel, ABI design does not really lend itself to mechanical testing, though. New interfaces simply need a lot of human review. That said, there is a place for unit tests; the kernel has been slow to adopt them, but that is beginning to change. Unit tests can detect regressions and unexpected changes, and help to ensure that a new interface lives up to what it is supposed to do. 

Consider, for example, the [`recvmmsg()`](<http://man7.org/linux/man-pages/man2/recvmmsg.2.html>) system call. Toward the end of the discussion before this call was merged, it gained a new `timeout` parameter. The expectation was that this timeout would apply to the call as a whole. In truth, it was only tested [![\[Michael Kerrisk\]](https://static.lwn.net/images/conf/2016/recipes/MichaelKerrisk-sm.jpg)](<https://lwn.net/Articles/704485/>) _after_ the receipt of a datagram; until something shows up, the timeout has no effect at all. In other words, nobody bothered to test it and, as a result, it was useless. 

Once tests are written, where should they go? The [Linux Test Project](<https://github.com/linux-test-project/ltp>) is the traditional home for such tests, but it is not ideal. It is an out-of-tree test suite, and new tests only show up there after the ABI they test has appeared in an official release. Test coverage is partial; in the end, it simply does not solve the problem. The kernel's [self-testing facility](<https://lwn.net/Articles/608959/>) is a better place; importantly, it has a paid maintainer. Those interested in working with kselftest can find more information on [the kernel.org wiki](<https://kselftest.wiki.kernel.org/>). 

There is only so much value to be had from testing, though, in the absence of a specification for how a new interface is expected to behave. In the case of `recvmmsg()`, nobody ever wrote that specification, so it was not possible to write a test for it. There are many benefits to written specifications; they serve as a target for the implementation and help those who write tests. A specification allows reviewers to critique the interface independently of the implementation, and increases the number of reviewers overall. This specification generally belongs in the changelog of the patch adding the new interface, though an even better approach is to send a man-page patch. 

The best thing to do, though, is to write a real application that uses the new interface. A while back he decided to delve into the [inotify](<http://man7.org/linux/man-pages/man7/inotify.7.html>) interface in order to improve its documentation. It is, in many ways, a good interface, much better than its predecessor. But it could have been better yet. At one point he thought he understood it, so he tried to write a real application that used it; the result was [this article series](<https://lwn.net/Articles/604686/>), among other things. 

That application required 1,500 lines of C code to get its job done. The inotify interface leaves a lot of work for the application to do. For example, change notifications lack user and process-ID information, making it impossible for a monitoring application to know who made a change. Directory monitoring is not recursive; if an application wants to watch a directory tree, it must set a separate watch on every directory in that tree. That, he said, may be unavoidable in the end. 

A problem that _was_ avoidable, instead, has to do with the renaming of files. A rename will generally result in two events; a "rename from" event and a "rename to" event. Unfortunately, these two events are not guaranteed to be consecutive in the event stream. In fact, they are not even guaranteed to both exist: if a file is renamed into a directory that is not monitored, the "rename to" event will not be generated. So an application has no definitive way to know if it will ever receive a "rename to" event or not; the result is a series of racy workarounds in user space. Life would have been much simpler if the two events had simply been guaranteed to show up together. 

The lesson, he said, is that the only way to find nontrivial ABI problems is to write real applications using the interface — before that interface is released to the world as a whole. 

Another way to improve our interfaces is to write documentation, of course. Describing what you're doing makes you think more deeply about it, he said. It also makes the new interface easier for others to understand, lowering the barriers to participation. A well-written man page is one way to do this documentation, but not the only way. 

#### Discovery and feedback

An ongoing problem area is discovery — there is no simple way to find out when a particular kernel ABI has changed. He doesn't have the time to follow everything on the linux-kernel list, and neither does anybody else. The linux-api list exists and should receive copies of patches that change interfaces, but that often fails to happen. So he relies on some scripts of his own to find changes, but they are imperfect. Often, interface changes are discovered by sheer luck when he stumbles across them. On rare occasion he'll actually get a man page for a new interface. 

He is far from the only person interested in interface changes. Application developers, C library developers, the `strace` maintainers, the Linux Test Project, and more all want to know about them. But user-space developers are typically the last to learn about changes — except in the unfortunate cases where even the kernel developers don't know that they changed something. Some changes to POSIX message queues in 3.5 broke the interface, for example. 2.6.12 featured an unexpected change to [`fcntl(F_SETOWN)`](<http://man7.org/linux/man-pages/man2/fcntl.2.html>) semantics. Nobody noticed until much later, at which point it was too late to fix things, since other programs depended on the new behavior. That is how we end up with options like `F_SETOWN_EX`, added in 2.6.32 in an attempt to fix the problems created by that change. 

That last example highlights a problem in the kernel's feedback loops. There are generally at least six months between when a new interface is added to the kernel and when users actually see it. In the worst case, design bugs will only be discovered when users start to look closely at this interface; by then, it is usually too late to fix them. We really need to get feedback sooner, before the kernel is committed to a specific interface. 

How do we get that feedback? Kerrisk's suggestions should not be surprising at this point: write a specification for the new feature, and write example programs that use it as well. Copy the patches liberally to the relevant mailing lists. Write documentation, or write an article for LWN. Don't just target kernel developers; publicize the details of the new interface broadly. Some developers, he said, have done all of these things, and the result has been far better interfaces. He called out Jeff Layton and his [open file description locks](<https://www.gnu.org/software/libc/manual/html_node/Open-File-Description-Locks.html>) ([formerly file-private locks](<https://lwn.net/Articles/586904/>)) work as an example of how to do it right. 

Is all of this overkill? Maybe, but it results in making a lot of people's lives easier. Especially his, he allowed, but not only his. By doing this work, developers can help to get more people involved in the process of looking at a new interface; that is necessary, since he alone does not scale well. The original developer has all of the information needed to judge a new interface; by getting it out there, they can bring more eyeballs to bear and have a much better chance of getting the interface right the first time. 

[Slides from and video of the talk](<https://kernel-recipes.org/en/2016/talks/man-pages-discovery-feedback-loops-and-the-perfect-kernel-commit-message/>) are available to those wanting more information. 

[Your editor thanks Kernel Recipes for assisting with his travel to the event.]
