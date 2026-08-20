---
title: EPOLL_CTL_DISABLE and multithreaded applications
url: https://lwn.net/Articles/520012/
date: "October 17, 2012"
category: "Epoll; User-space API"
author: "By Michael Kerrisk October 17, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Michael Kerrisk**  
October 17, 2012

Other than the merging of the server-side component of [TCP Fast Open](<https://lwn.net/Articles/508865/>), one of the few user-space API changes that has gone into the just-closed 3.7 merge window is the addition of a new `EPOLL_CTL_DISABLE` operation for the `epoll_ctl()` system call. It's interesting to look at this operation as an illustration of the sometimes unforeseen complexities of dealing with multithreaded applications; that examination is the subject of this article. However, the addition of the `EPOLL_CTL_DISABLE` feature highlights some common problems in the design of the APIs that the kernel presents to user space. (To be clear: `EPOLL_CTL_DISABLE` is the _fix_ to a past design problem, not a design problem itself.) These design problems will be the subject of a follow-on article next week. 

Understanding the need for `EPOLL_CTL_DISABLE` requires an understanding of several features of the epoll API. For those who are unfamiliar with epoll, we begin with a high-level picture of how the API works. We then look at the problem that `EPOLL_CTL_DISABLE` is designed to solve, and how it solves that problem. 

####  An overview of the epoll API

The (Linux-specific) epoll API allows an application to monitor multiple file descriptors in order to determine which of the descriptors are ready to perform I/O. The API was designed as a more efficient replacement for the traditional `select()` and `poll()` system calls. Roughly speaking, the performance of those older APIs scales linearly with the number of file descriptors being monitored.  That behavior makes `select()` and `poll()` poorly suited for modern network applications that may handle thousands of file descriptors simultaneously. 

The poor performance of `select()` and `poll()` is an inescapable consequence of their design. For each monitoring operation, both system calls require the application to give the kernel a complete list of all of the file descriptors that are of interest. And on each call, the kernel must re-examine the state of all of those descriptors and then pass a data structure back to the application that describes the readiness of the descriptors. 

The underlying problem of the older APIs is that they don't allow an application to inform the kernel about its _ongoing_ interest in a (typically unchanging) set of file descriptors. If the kernel had that information, then, as each file descriptor became ready, it could record the fact in preparation for the next request by the application for the set of ready file descriptors. The epoll API allows exactly that approach, by splitting the monitoring API up across three system calls: 

  * `epoll_create()` creates an internal kernel data structure ("an epoll instance") that is used to record the set of file descriptors that the application is interested in monitoring. The call returns a file descriptor that is used in the remaining epoll APIs. 

  * `epoll_ctl()` allows the application to inform the kernel about the set of file descriptors it would like to monitor by adding (`EPOLL_CTL_ADD`) and removing (`EPOLL_CTL_DEL`) file descriptors from the _interest list_ of the epoll instance. `epoll_ctl()` can also modify (`EPOLL_CTL_MOD`) the set of events that are to be monitored for a file descriptor that is already in the interest list. Once a file descriptor has been recorded in the interest list, the kernel tracks I/O events for the file descriptor (e.g., the arrival of new input); if the event causes the file descriptor to become ready, the kernel places the descriptor on the _ready list_ of the epoll instance, in preparation for the next call to `epoll_wait()`. 

  * `epoll_wait()` requests the kernel to return one or more ready file descriptors. The kernel satisfies this request by simply fetching items from the ready list (the call can block if there are no descriptors that are yet ready). The application uses `epoll_wait()` each time it wants to check for changes in the readiness of file descriptors. What is notable about `epoll_wait()` is that the application does not need to pass in a list of file descriptors on each call: the kernel already has that information via preceding calls to `epoll_ctl()`. In addition, there is no need to rescan the complete set of file descriptors to see which are ready; the kernel has already been recording that information on an ongoing basis because it knows which file descriptors the application is interested in. 

Schematically, the epoll API operates as shown in the following diagram: 

> ![\[Overview of the epoll
API\]](https://static.lwn.net/images/2012/epoll_API.png)

Because the kernel is able to maintain internal state about the set of file descriptors in which the application is interested, `epoll_wait()` is much more efficient than `select()` and `poll()`. Roughly speaking, its performance scales according to the number of ready file descriptors, rather than the total number of file descriptors being monitored. 

#### Epoll and multithreaded applications: the problem

The author of the patch that implements `EPOLL_CTL_DISABLE`, Paton Lewis, is not a regular kernel hacker. Rather, he's a developer with a particular user-space itch, and it would seem that a kernel change is the only way of scratching that itch. In the description accompanying the first iteration of his patch, Paton began with the following [observation](<https://lwn.net/Articles/520022/>): 

It is not currently possible to reliably delete epoll items when using the same epoll set from multiple threads. After calling epoll_ctl with EPOLL_CTL_DEL, another thread might still be executing code related to an event for that epoll item (in response to epoll_wait). Therefore the deleting thread does not know when it is safe to delete resources pertaining to the associated epoll item because another thread might be using those resources. 

The deleting thread could wait an arbitrary amount of time after calling epoll_ctl with EPOLL_CTL_DEL and before deleting the item, but this is inefficient and could result in the destruction of resources before another thread is done handling an event returned by epoll_wait. 

The fact that the kernel records internal state is the source of a complication for multithreaded applications. The complication arises from the fact that applications may also want to maintain state information about file descriptors. One possible reason for doing this is to prevent file descriptor starvation, the phenomenon that can occur when, for example, an application determines that a file descriptor has data available for reading and then attempts to read all of the available data. It could happen that there is a very large amount of data available (for example, another application may be continuously writing data on the other end of a socket connection). Consequently, the reading application would be tied up for a long period; meanwhile, it does not service I/O events on the other file descriptors--those descriptors are starved of service by the application. 

The solution to file descriptor starvation is for the application to maintain a user-space data structure that caches the readiness of each of the file descriptors that it is monitoring. Whenever `epoll_wait()` informs the application that a file descriptor is ready, then, instead of performing as much I/O as possible on the descriptor, the application makes a record in its cache that the file descriptor is ready. The application logic then takes the form of a loop that (a) periodically calls `epoll_wait()` and (b) performs a _limited_ amount of I/O on the file descriptors that are marked as ready in the user-space cache. (When the application finds that I/O is no longer possible on one of the file descriptors, then it can mark that descriptor as not ready in the cache.) 

Thus, we have a scenario where the both kernel and a user-space application are maintaining state information about the same resources. This can potentially lead to race conditions when competing threads in a multithreaded application want to update state information in both places. The most fundamental piece of state information maintained in both places is "existence". 

For example, suppose that an application thread determines that it is no longer necessary to monitor a file descriptor. The thread would first check to see whether the file descriptor is marked as ready in the user-space cache (i.e., there may still be some outstanding I/O to perform), and then, if the file descriptor is not ready, the thread would delete the file descriptor from the user-space cache and from the kernel's epoll interest list using the `epoll_ctl(EPOLL_CTL_DEL)` operation. However, these steps could fall afoul in scenarios such as the following involving two threads operating on file descriptor 9: 

> **Thread 1** | **Thread 2**  
> ---|---  
> Determine from the user-space cache that descriptor 9 is not ready.  |   
> |  Call `epoll_wait()`; the call indicates descriptor 9 as ready.   
> |  Record descriptor 9 as being ready inside the user-space cache so that I/O can later be performed.   
> Delete descriptor 9 from the user-space cache.  |   
> Delete descriptor 9 from the kernel's epoll interest list using `epoll_ctl(EPOLL_CTL_DEL)`.  |   
  
Following the above scenario, some data will be lost. Other scenarios could lead to a corrupted cache or an application crash. 

No use of (per-file-descriptor) mutexes can eliminate the sorts of races described here, short of protecting the calls to `epoll_wait()` with a (global) mutex, which has the effect of destroying concurrency. (If one thread is blocked in a `epoll_wait()` call, then any other thread that tries to acquire the corresponding mutex will also block.) 

#### Epoll and multithreaded applications: the solution

Paton's solution to this problem is to extend the epoll API with a new operation that atomically prevents other threads from receiving further indications that a file descriptor is ready, while at the same time informing the caller whether another thread has "recently" been told the file descriptor is ready. The new operation relies on some of the inner workings of the epoll API. 

When adding (`EPOLL_CTL_ADD`) or modifying (`EPOLL_CTL_MOD`) a file descriptor in the interest list, the application specifies a mask of I/O events that are of interest for the descriptor. For example, the mask might include both `EPOLLIN` and `EPOLLOUT`, if the application wants to know when the file descriptor becomes either readable or writable. In addition, the kernel implicitly adds two further flags to the events mask in the interest list: `EPOLLERR`, which requests monitoring for error conditions, and `EPOLLHUP`, which requests monitoring for a "hangup" (e.g., we are monitoring the read end of a pipe, and the write end is closed). When a file descriptor becomes ready, `epoll_wait()` returns a mask that contains all of the requested events for which the file descriptor is ready. For example, if an application requests monitoring of the read end of a pipe using `EPOLLIN` and the write end of the pipe is closed, then `epoll_wait()` will return an events mask that includes both `EPOLLIN` and `EPOLLHUP`. 

As well as the flags that can be used to monitor file descriptors for various I/O events, there are a few "operational flags"--flags that modify the semantics of the monitoring operation itself. One of these is `EPOLLONESHOT`. If this flag is specified in the events mask for a file descriptor, then, once the file descriptor becomes ready and is returned by a call to `epoll_wait()`, it is disabled from further monitoring (but remains in the interest list). If the application is interested in monitoring file descriptor once more, then it must re-enable the file descriptor using the `epoll_ctl(EPOLL_CTL_MOD)` operation. 

> Per-descriptor events mask recorded in an epoll interest list   
> ---  
> Operational flags  |  I/O event flags   
> `EPOLLONESHOT`, `EPOLLET`, ...  |  `EPOLLIN`, `EPOLLOUT`, `EPOLLHUP`, `EPOLLERR`, ...   
  
The implementation of `EPOLLONESHOT` relies on a trick. If this flag is set, then, if the file descriptor indicates as being ready via `epoll_wait()`, the kernel clears all of the "non-operational flags" (i.e., the I/O event flags) in the events mask for that file descriptor. This serves as a later cue to the kernel that it should not track I/O events for this file descriptor. 

By now, we finally have enough details to understand Paton's extension to the epoll API--the `epoll_ctl(EPOLL_CTL_DISABLE)` operation--that allows multithreaded applications to avoid the kind of races described above. To successfully use this extension requires the following: 

  1. The user-space cache that describes file descriptors should also include a per-descriptor "delete-when-done" flag that defaults to false but can be set true when one thread wants to inform another thread that a particular file descriptor should be deleted. 

  2. All `epoll_ctl()` calls that add or modify file descriptors in the interest list must specify the `EPOLLONESHOT` flag. 

  3. The `epoll_ctl(EPOLL_CTL_DISABLE)` operation should be used as described in a moment. 

In addition, calls to `epoll_ctl(EPOLL_CTL_DISABLE)` and accesses to the user-space cache must be suitably protected with per-file-descriptor mutexes. We won't go into details here, but [the second version](<https://lwn.net/Articles/520019/>) of Paton's patch adds a sample application to the kernel source tree (under `tools/testing/selftests/epoll/test_epoll.c`) that demonstrates the principles. 

The new epoll operation is employed via the following call: 

```
epoll_ctl(epfd, EPOLL_CTL_DISABLE, fd, NULL);
```

`epfd` is a file descriptor referring to an epoll instance. `fd` is the file descriptor in the interest list that is to be disabled. The semantics of this operation handle two cases: 

  * One or more of the I/O event flags is set in the interest list entry for `fd`. This means that, since the last `epoll_ctl()` operation that added or modified this interest list entry, no other thread has executed an `epoll_wait()` call that indicated this file descriptor as being ready. In this case, the kernel clears the I/O event flags in the interest list entry, which prevents subsequent `epoll_wait()` calls from returning the file descriptor as being ready. The `epoll_ctl(EPOLL_CTL_DISABLE)` call then returns zero to the caller. At this point, the caller knows that no other thread is operating on the file descriptor, and it can thus safely delete the descriptor from the user-space cache and from the kernel interest list. 

  * No I/O event flag is set in the interest list entry for `fd`. This means that since the last `epoll_ctl()` operation that added or modified this interest list entry, another thread has executed an `epoll_wait()` call that indicated this file descriptor as being ready. In this case, `epoll_ctl(EPOLL_CTL_DISABLE)` returns -1 with `errno` set to `EBUSY`. At this point, the caller knows that another thread is operating on the descriptor, so it sets the descriptor's "delete-when-done" flag in the user-space cache to indicate that the other thread should delete the file descriptor once when it has finished using it. 

Thus, we see that with a moderate amount of effort, and a little help from a new kernel interface, a race can be avoided when deleting file descriptors in multithreaded applications that wish to avoid file descriptor starvation. 

#### Concluding remarks

There was relatively little comment on the first iteration of Paton's patch. The only substantive comments came from Christof Meerwald; in response to these, Paton created the second version of his patch. That version received _no_ comments, and was incorporated into 3.7-rc1. It would be nice to think that the relatively paucity of comments reflects the silent agreement that Paton's approach is correct. However, one is left with the nagging feeling that in fact few people have reviewed the patch, which leaves open the question: is this the best solution to the problem? 

Although `EPOLL_CTL_DISABLE` solves the problem, the solution is neither intuitive nor easy to use. The main reason for this is that `EPOLL_CTL_DISABLE` is a bolt-on hack to the epoll API that satisfies the requirement (often [repeated](<https://lwn.net/Articles/486395/>) by Linus Torvalds) that _existing_ user-space applications must not be broken by making a kernel ABI change. Within that constraint, `EPOLL_CTL_DISABLE` _may_ be the best solution to the problem. However, it seems certain that a better solution might have been possible if it had incorporated during the _original_ design of the epoll API. Next week's follow-on article will consider whether a better initial solution could have been found and also consider why it might not be possible to find a better solution within the constraints of the current API. 

Finally, it's worth noting that the `EPOLL_CTL_DISABLE` feature is not yet cast in stone, although it will become so in about two months, when Linux 3.7 is released. In the meantime, if someone comes up with a better idea to solve the problem, then the existing approach could be modified or replaced.
