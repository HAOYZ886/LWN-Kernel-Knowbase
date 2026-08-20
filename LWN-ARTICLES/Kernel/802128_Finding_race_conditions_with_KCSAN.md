---
title: Finding race conditions with KCSAN
url: https://lwn.net/Articles/802128/
date: "October 14, 2019"
category: "Development tools-Sanitizers"
author: "By Jonathan Corbet October 14, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 14, 2019

Race conditions can be some of the trickiest bugs to find. The resulting problems can be subtle, and reproducing the problem in order to track it down can be difficult or impossible; often code inserted to narrow down a race condition will cause it to stop manifesting entirely. A tool that can find race conditions automatically would thus be a valuable thing for the kernel community to have. In late September, Marco Elver [announced a tool called KCSAN](<https://lwn.net/ml/linux-kernel/CANpmjNPJ_bHjfLZCAPV23AXFfiPiyXXqqu72n6TgWzb2Gnu1eA@mail.gmail.com/>) (the Kernel Concurrency Sanitizer) that does exactly that — and which has already found a number of real problems. 

A common cause of race conditions is unserialized access to shared data leading to inconsistent views of the state of the world. Such problems can be hard to identify with code inspection, since they involve the interaction of multiple sites within the code itself and generally require a rare type of bad timing. But an automated tool might be able to note when this kind of access happens and flag a problem even if the system manages not to misbehave as a result. KCSAN finds potential problems by monitoring access to memory locations and identifying patterns where: 

  * multiple threads of execution access the location, 
  * those accesses are unordered — not protected by a lock, for example, and, 
  * at least one of those accesses is a write. 

Needless to say, watching every memory location that the kernel accesses would be a massive task, so KCSAN takes a statistical approach in the hope of stumbling across problems eventually. 

#### How it works

The first step is to compile the kernel with the `-fsanitize=thread` option, which is supported by both GCC and Clang. This will cause the compiled code to be instrumented to allow the monitoring of its memory accesses. Specifically, each memory access will be augmented by a function call; if the program reads a four-byte quantity at `addr`, for example, the generated code will first make a call to `__tsan_read4(addr)`. The monitoring code provides these `__tsan_readN()` and `__tsan_writeN()` functions, which can then do something useful with the access pattern it sees. 

In the case of KCSAN, these function calls are simply ignored 1,999 out of 2,000 times; to do otherwise would slow the kernel to a point of complete unusability. On the 2,000th time, though, KCSAN keeps an eye on the address for a period of time, looking for other accesses. While running in the context of the thread where the access was performed, KCSAN will set a "watchpoint", which is done by recording the address, the size of the data access, and whether the access was a write in a small table. This thread will then simply delay for (by default) 10µs. 

The above picture is simplified somewhat; there are a couple of exceptions to keep in mind. The first of those is that, before deciding whether to ignore an access, KCSAN looks to see if there is already a watchpoint established for the address in question. If so, and if either the current access or the access that created the watchpoint is a write, then a race condition has been detected and a report will be sent to the system log. 

In the absence of a watch point, the code will check whether the current access is being performed in an atomic context (using KCSAN's definition, which is a bit different than what the rest of the kernel uses) before deciding whether to ignore the access or not. An atomic access, thus, will not result in the creation of a watch point, but if one already exists then the code is accessing the data location in question in both atomic and non-atomic ways, which rarely leads to good things. 

Meanwhile, the original thread is delaying after having set the watchpoint. At the end of the delay period, the watchpoint will be deleted and monitoring of that address stops. But before execution continues, the value at the accessed address will be checked; if it has changed since the watchpoint was set, a race condition is once again deemed to have occurred. 

Naturally, the above story leaves out some details, but that is the core of the algorithm used. One would expect it to miss a lot of races, since it is only looking at a fraction of the kernel's memory accesses and only watches any given location for a short period of time. But, if run for long enough, KCSAN does indeed appear to be able to find race conditions that have escaped the developers of the code in question. 

#### KCSAN in action

The syzbot system has started adding KCSAN to its fuzzing runs, resulting in some immediate bug reports. The [first of those](<https://lwn.net/ml/linux-kernel/0000000000009b403005942237bf@google.com/>) was in the kernel's "taskstats" subsystem; that resulted in [a fix](<https://lwn.net/ml/linux-kernel/20191009114809.8643-1-christian.brauner@ubuntu.com/>) from Christian Brauner that has not yet made it into the mainline. After five revisions, though, chances are that it should be close to being ready. 

Few parts of the kernel have been more closely scrutinized for concurrency issues than the read-copy-update (RCU) subsystem, which is itself a concurrency-management mechanism. On October 7, KCSAN [found a data race in RCU](<https://lwn.net/ml/linux-kernel/000000000000604e8905944f211f@google.com/>); various patches have been circulated to fix it but, once again, nothing has found its way to the mainline yet. 

As might be expected, KCSAN also produces warnings in situations that are not actual race conditions. Silencing those warnings requires annotating the code, often by marking the accesses in question with `READ_ONCE()` or `WRITE_ONCE()`. Some developers have already raised concerns that KCSAN reports will lead to a flurry of "fixes" from developers who may not fully understand the code they are working with. Even false positives can have their value, though; while [addressing some reports in the TCP code](<https://lwn.net/ml/netdev/20191011031746.16220-1-edumazet@google.com/>), Eric Dumazet discovered and fixed a real race that was added with the [TCP fast open](<https://lwn.net/Articles/508865/>) mechanism in 2012. 

KCSAN is a new tool that still has some rough edges to it. Some time will be required to smooth those down and for the development community as a whole to figure out how to integrate its reports into the development process. There is clear value, though, in a tool that can find this kind of obscure problem; it should lead to more reliable kernels in the future.
