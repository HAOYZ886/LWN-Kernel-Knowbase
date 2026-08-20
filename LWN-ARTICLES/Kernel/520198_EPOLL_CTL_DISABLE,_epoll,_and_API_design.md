---
title: "EPOLL_CTL_DISABLE, epoll, and API design"
url: https://lwn.net/Articles/520198/
date: "October 23, 2012"
category: "Epoll; User-space API-Design"
author: "By Michael Kerrisk October 23, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Michael Kerrisk**  
October 23, 2012

[In an article last week](<https://lwn.net/Articles/520012/>), we saw that the `EPOLL_CTL_DISABLE` operation proposed by Paton Lewis provides a way for multithreaded applications that cache information about file descriptors to safely delete those file descriptors from an epoll interest list. For the sake of brevity, in the remainder of this article we'll use the term "the `EPOLL_CTL_DISABLE` problem" to label the underlying problem that `EPOLL_CTL_DISABLE` solves. 

This article revisits the `EPOLL_CTL_DISABLE` story from a different angle, with the aim of drawing some lessons about the design of the APIs that the kernel presents to user space. The initial motivation for pursuing this angle arises from the observation that the `EPOLL_CTL_DISABLE` solution has some difficulties of its own. It is neither intuitive (it relies on some non-obvious details of the epoll implementation) nor easy to use. Furthermore, the solution is somewhat limiting, since it forces the programmer to employ the `EPOLLONESHOT` flag. Of course, these difficulties arise at least in part because `EPOLL_CTL_DISABLE` is designed so as to satisfy one of the cardinal rules of Linux development: interface changes must not break existing user-space applications. 

If there had been an awareness of the `EPOLL_CTL_DISABLE` problem when the epoll API was originally designed, it seems likely that a better solution would have been built, rather than bolting on `EPOLL_CTL_DISABLE` after the fact. Leaving aside the question of what that solution might have been, there's another interesting question: could the problem have been foreseen? 

One might suppose that predicting the `EPOLL_CTL_DISABLE` problem would have been quite difficult. However, the synchronized-state problem is well known and the epoll API was designed to be thread friendly. Furthermore, the notion of employing a user-space cache of the ready list to prevent file descriptor starvation was documented in the [`epoll(7)` man page](<http://man7.org/linux/man-pages/man7/epoll.7.html>) (see the sections "Example for Suggested Usage" and "Possible Pitfalls and Ways to Avoid Them") that was supplied as part of the original implementation. 

In other words, almost all of the pieces of the puzzle were known when the epoll API was designed. The one fact whose implications _might_ not have been clear was the presence of a blocking interface (`epoll_wait()`) in the API. One wonders if more review (and building of test applications) as the epoll API was being designed might have uncovered the interaction of `epoll_wait()` with the remaining well-known pieces of the puzzle, and resulted in a better initial design that addressed the `EPOLL_CTL_DISABLE` problem. 

So, the first lesson from the `EPOLL_CTL_DISABLE` story is that more review is necessary in order to create better API designs (and we'll see further evidence supporting that claim in a moment). Of course, the need for more review is a general problem in all aspects of Linux development. However, the effects of insufficient review can be especially painful when it comes to API design. The problem is that once an API has been released, applications come to depend on it, and it becomes at the very least difficult, or, more likely, impossible to later change the aspects of the API's behavior that applications depend upon. As a consequence, a mistake in API design by one kernel developer can create problems that thousands of user-space developers must live with for many years. 

A second lesson about API design can be found in [a comment](<https://lwn.net/Articles/520168/>) that Paton made when responding to a question from Andrew Morton about the design of `EPOLL_CTL_DISABLE`. Paton was speculating about whether a call of the form: 

```
epoll_ctl(epfd, EPOLL_CTL_DEL, fd, &epoll_event);
```

could be used to provide the required functionality. The `EPOLL_CTL_DEL` operation does not currently use the fourth argument of `epoll_ctl()`, and applications should specify it as `NULL` (but more on that point in a moment). The idea would be that ""epoll_ctl [EPOLL_CTL_DEL] could set a bit in epoll_event.events (perhaps called EPOLLNOTREADY)"" to notify the caller that the file descriptor was in use by another thread. 

But Paton noted a shortcoming of this approach: 

However, this could cause a problem with any legacy code that relies on the fact that the epoll_ctl epoll_event parameter is ignored for EPOLL_CTL_DEL. Any such code which passed an invalid pointer for that parameter would suddenly generate a fault when running on the new kernel code, even though it worked fine in the past. 

In other words, although the `EPOLL_CTL_DEL` operation doesn't use the `epoll_event` argument, the caller is _not_ required to specify it as `NULL`. Consequently, existing applications are free to pass random addresses in `epoll_event`. If the kernel now started using the `epoll_event` argument for `EPOLL_CTL_DEL`, it seems likely that some of those applications would break. Even though those applications might be considered poorly written, that's no justification for breaking them. [Quoting](<http://thread.gmane.org/gmane.linux.kernel/361522/focus=361828>) Linus Torvalds: 

We care about user-space interfaces to an insane degree. We go to extreme lengths to maintain even badly designed or unintentional interfaces. Breaking user programs simply isn't acceptable. 

The lesson here is that when an API doesn't use an argument, usually the right thing to do is for the implementation to include a check that requires the argument to have a suitable "empty" value, such as `NULL` or zero. Failure to do that means that we may later be prevented from making the kind of API extensions that Paton was talking about. (We can leave aside the question of whether this particular extension to the API was the right approach. The point is that the option to pursue this approach was unavailable.) The kernel-user-space API provides numerous examples of failure to do this sort of checking. 

However, there is yet more life in this story. Although there have been many examples of system calls that failed to check that "empty" values were passed for unused arguments, it turns out that `epoll_ctl(EPOLL_CTL_DEL)` fails to include the check for another reason. Quoting the BUGS section of the [`epoll_ctl()` man page](<http://man7.org/linux/man-pages/man2/epoll_ctl.2.html#BUGS>): 

In kernel versions before 2.6.9, the `EPOLL_CTL_DEL` operation required a non-NULL pointer in `event` [the `epoll_event` argument], even though this argument is ignored. Since Linux 2.6.9, `event` can be specified as NULL when using `EPOLL_CTL_DEL`. Applications that need to be portable to kernels before 2.6.9 should specify a non-NULL pointer in event. 

In other words, applications that use `EPOLL_CTL_DEL` are not only _permitted_ to pass random values in the `epoll_event` argument: if they want to be portable to Linux kernels before 2.6.9 (which [fixed](<http://thread.gmane.org/gmane.linux.kernel.commits.head/40275>) the problem), they are _required_ to pass a pointer to some random, but valid user-space address. (Of course, most such applications would simply allocate an unused `epoll_event` structure and pass a pointer to that structure.) Here, we're back to the first lesson: more review of the initial epoll API design would almost certainly have uncovered this fairly basic design error. (It's this writer's contention that one of the best ways to conduct that sort of review is by thoroughly documenting the API, but he admits to a [certain bias](<https://lwn.net/Articles/247788/>) on this point.) 

Failing to check that unused arguments (or unused pieces of arguments) have "empty" values can cause subtle problems long after the fact. Anyone looking for further evidence on that point does not need to go far: the `epoll_ctl()` system call provides another example. 

Linux 3.5 added a new epoll flag, `[EPOLLWAKEUP](<https://lwn.net/Articles/495473/>)`, that can be specified in the `epoll_event.events` field passed to `epoll_ctl()`. The effect of this flag is to prevent the system from being suspended while epoll readiness events are pending for the corresponding file descriptor. Since this flag has a system-wide effect, the caller must have a capability, `CAP_BLOCK_SUSPEND` ([initially misnamed `CAP_EPOLLWAKEUP`](<https://lwn.net/Articles/520770/>)). 

In the initial `EPOLLWAKEUP` implementation, if the caller did not have the `CAP_BLOCK_SUSPEND` capability, then `epoll_ctl()` returned an error so that the caller was informed of the problem. However, Jiri Slaby [reported](<https://lwn.net/Articles/520774/>) that the new flag caused a regression: an existing program failed because it was setting formerly unused bits in `epoll_event.events` when calling `epoll_ctl()`. When one of those bits acquired a meaning (as `EPOLLWAKEUP`), the call failed because the program lacked the required capability. The problem of course is that `epoll_ctl()` has never checked the flags in `epoll_event.events` to ensure that the caller has specified only flag bits that are actually implemented in the kernel. Consequently, applications were free to pass random garbage in the unused bits. 

When one of those random bits suddenly caused the application to fail, what should be done? Following the logic outlined above, of course the answer is that the kernel must change. And that is exactly what happened in this case. [A patch](<https://lwn.net/Articles/520775/>) was applied so that if the `EPOLLWAKEUP` flag was specified in a call to `epoll_ctl()` and the caller did not have the `CAP_BLOCK_SUSPEND` capability, then `epoll_ctl()` _silently ignored_ the flag instead of returning an error. Of course, in this case, the calling application might easily carry on, unaware that the request for `EPOLLWAKEUP` semantics had been ignored. 

One might observe that there is a certain arbitrariness about the approach taken to dealing with the `EPOLLWAKEUP` breakage. Taken to the extreme, this type of logic would say that the kernel can never add new flags to APIs that didn't hitherto check their bit-mask arguments--and there is a long list of such system calls (`mmap()`, `splice()`, and `timer_settime()`, to name just a few). Nevertheless, new flags are added. So, for example, Linux 2.6.17 added the epoll event flag `EPOLLRDHUP`, and since no one complained about a broken application, the flag remained. It seems likely that the same would have happened for the original implementation of `EPOLLWAKEUP` that returned an error when `CAP_BLOCK_SUSPEND` was lacking, if someone hadn't chanced to make an error report. 

As an aside to the previous point, in cases where someone reports a regression _after_ an API change has been officially released, there is a conundrum. On the one hand, there may be old applications that depend on the previous behavior; on the other hand, newer applications may already depend on the newly implemented change. At that point, there is no simple remedy: to fix things almost certainly means that some applications must break. 

We can conclude with two observations, one specific, and the other more general. The specific observation is that, ironically, `EPOLL_CTL_DISABLE` itself seems to have had surprisingly little review before being accepted into the 3.7 merge window. And in fact, now that more attention has been focused on it, it [looks as though](<https://lwn.net/Articles/520358/>) the proposed API will see some changes. So, we have a further, very current, piece of evidence that there is still insufficient review of kernel-user-space APIs. 

More generally, the problem seems to be that--while the kernel code gets reviewed on many dimensions--it is relatively uncommon for kernel-user-space APIs to be reviewed on their own merits. The kernel has maintainers for many subsystems. By now, the time seems ripe for there to be a kernel-user-space API maintainer--someone whose _job_ it is to actively review and ack every kernel-user-space API change, and to ensure that test cases and sufficient documentation are supplied with the implementation of those changes. Lacking such a maintainer, it seems likely that we'll see many more cases where kernel developers add badly designed APIs that cause [years of pain [PDF]](<http://man7.org/conf/lcna2012/Why_kernel_space_sucks-2012-08-29.pdf>) for user-space developers.
