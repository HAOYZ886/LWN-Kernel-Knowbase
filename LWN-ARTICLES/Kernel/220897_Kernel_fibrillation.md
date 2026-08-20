---
title: Kernel fibrillation
url: https://lwn.net/Articles/220897/
date: "February 6, 2007"
category: Asynchronous IO; Fibrils
author: 
---

[Last week's article on fibrils](<http://lwn.net/Articles/219954/>) caught the discussion in a relatively early state. That discussion is _still_ in an early state, but some interesting ground has been covered. Here, we'll catch up on a few themes from that conversation. 

Alan Cox has [requested that the "fibril" name be dumped](<https://lwn.net/Articles/220899/>): 

The constructs Zach is using appear to be identical to co-routines, and they've been called that in computer science literature for fifty years. They are one of the great and somehow forgotten ideas. 

Alan also points out that a number of hazards lie between the current state of the fibril patch and anything robust enough for the mainline kernel - but everybody involved already knew that. Linus [acknowledges](<https://lwn.net/Articles/220901/>) the similarities with coroutines, but also maintains that they are sufficiently different to merit their own name. A full coroutine implementation in the kernel, he says, would be impractical. 

Linus has also [responded](<https://lwn.net/Articles/220902/>) to Ingo Molnar's criticisms of the fibril concept. He maintains that the real benefits to fibrils are (1) the elimination of the separate code paths currently associated with asynchronous I/O, and (2) reductions in setup and teardown costs. The latter is significant, he says, because the bulk of asynchronous operations can actually be satisfied from cache; being able to run those operations without going through the full AIO setup would be a big win. 

Ingo has clarified his comments somewhat. The stumbling point seems to be the addition of a new scheduling concept which, he thinks, is not necessary. He has proposed alternatives which take the form of a pool of kernel threads; rather than create a fibril, a blocking system call could simply switch to another kernel thread which is there waiting for just that occasion. Ingo believes that kernel threads perform well enough to handle this task, and they could be made lighter; in addition, the use of kernel threads would allow asynchronous calls to spread across a multi-CPU system. Fibrils, instead, are currently limited to a single processor. Zach Brown, the creator of the fibril patchset, seems to think that the idea is at least worth a try. Linus, instead, has [said](<https://lwn.net/Articles/220908/>) that any adaptation of kernel threads to this task would end up looking a lot like fibrils anyway. Rather than bear the expense of keeping a (potentially large) pool of kernel threads around, one might as well just create a truly lightweight object - a fibril. 

Some discussion of the eventual user-space API has occurred. Linus has [suggested](<https://lwn.net/Articles/220909/>) that the asynchronous submission call look something like this: 

```
long async_submit(unsigned long flags, long *result_pointer,
                          long syscall_number, unsigned long *args);
```

The role of the `flags` argument has not really been discussed; one just assumes such an argument will be necessary, sooner or later. The `result_pointer` argument tells the kernel where to put the result of the operation. Interestingly, the result code would follow the in-kernel conventions: zero for success or a negative error code for failure. While the operation is outstanding, the kernel would store a positive "cookie" value which could be used by the application to wait for (or cancel) the call. 

The `wait_for_async()` system call remains for applications wanting to get the completion status of their asynchronous operations. There have been a couple of requests, however, for a mechanism by which applications could obtain completion status without having to go back into the kernel. That inspired David Miller to [complain](<https://lwn.net/Articles/220912/>) about a big part of the conversation which is _not_ happening: the integration with the [kevent patches](<http://lwn.net/Articles/213672/>). Much of the kevent work has been aimed at solving just this problem, but Evgeniy Polyakov continues to have trouble getting people to look at it. To a great extent, `wait_for_async()` is another event interface. It seems unlikely that the kernel needs two of them. 

What does all this work bode for the existing asynchronous I/O interface, and, in particular, the [buffered filesystem AIO patches](<http://lwn.net/Articles/216200/>) which have not yet been merged? Seeking to fend off doubt about the future of that interface, Suparna Bhattacharya has [argued that the buffered AIO patches should still be merged](<https://lwn.net/Articles/220914/>): 

Since this is going to be a new interface, not the existing linux AIO interface, I do not see any conflict between the two. Samba4 already uses fsaio, and we now have the ability to do POSIX AIO over kernel AIO (which depends on fsaio). The more we delay real world usage the longer we take to learn about the application patterns that matter. And it is those patterns that are key. 

Decision time will be soon, since the buffered AIO patches seem to be ready for merging into 2.6.21. Over the next couple of weeks, somebody will have to decide whether to merge those patches - and maintain them indefinitely - or hold off with the idea that fibrils will evolve into the preferred solution. 

Finally, Bert Hubert [noted](<https://lwn.net/Articles/220920/>) that DragonFly BSD had an asynchronous system call interface - until last July, when the developers pulled it out. DragonFly had created two system calls - `sendsys2()` and `waitsys2()` \- which split up the tasks of initiating a system call and waiting for its completion. A [followup](<https://lwn.net/Articles/220921/>) suggests that DragonFly BSD had taken a different approach, requiring that every system call have asynchronous support built into it. In that sense, their asynchronous interface looked like a more general version of Linux AIO. 

Pushing asynchronous support down into system calls, filesystems, and device drivers brings a lot of complexity; the slow progress of Linux AIO illustrates just how hard it can be. One of the major advantages of the fibril idea is that (with few exceptions) the system calls do not have to be changed; they do not need to be aware of asynchronous operation at all. The ability to pull asynchronous support into a relatively small chunk of core kernel code may be the key idea that sells the entire fibril concept.
