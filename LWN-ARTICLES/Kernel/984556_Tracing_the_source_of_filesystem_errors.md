---
title: Tracing the source of filesystem errors
url: https://lwn.net/Articles/984556/
date: "August 7, 2024"
category: Filesystems
author: "By Jake Edge August 7, 2024 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
August 7, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

There are lots of places in the kernel where an `EINVAL` can be returned to user space, but it is often unclear what the actual underlying problem is because the [`errno`](<https://man7.org/linux/man-pages/man3/errno.3.html>) error codes are too generic. That is the problem that Miklos Szeredi wanted to discuss in a filesystem session that he led remotely at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). He would like to help those who are trying to debug problems trace where in the kernel a particular error code is being generated. 

Filesystem mounting is an example of where this problem can occur, Szeredi said; there are lots of places where `EINVAL` is returned, so it does not really tell anyone anything. If he is debugging a kernel filesystem and receives an error, he wants to know where in the code that occurred. The [`strace`](<https://man7.org/linux/man-pages/man1/strace.1.html>) tool is useful for debugging, so ideally whatever is done to help show where errors are coming from would integrate with it. 

He does not think it would be difficult to add something along those lines, though it would be best if any solution did not require root privileges. He mentioned the existing solutions, which includes messages in dmesg, but those are "not ideal for debugging". For filesystems that use the [new mount API](<https://lwn.net/Articles/979166/>), there is a file descriptor returned from `fsopen()` that can be used to read error messages. 

Another possibility is using [ftrace](<https://docs.kernel.org/trace/ftrace.html>), which can be used to trace the source of certain errors, but the tracefs interface is difficult to use, he said. The [`trace-cmd`](<https://man7.org/linux/man-pages/man1/trace-cmd.1.html>) tool is a more user-friendly interface, but it does not yet support the `funcgraph-retval` option of the [function_graph tracer](<https://docs.kernel.org/trace/ftrace.html#function-graph-tracer>), which is the way to get the information needed. It would also be nice if `trace-cmd` had an option to filter on negative return values, which should be easy to add. But it needs root privileges and has a global scope, which makes it difficult to integrate with `strace`. 

He explored what an `strace`-friendly solution might look like. He suggested that an error descriptor could be added to [`struct task_struct`](<https://elixir.bootlin.com/linux/v6.10.3/source/include/linux/sched.h#L748>) so that when errors occurred, the code could use `current->err_desc` to store a string with information about where the error was generated. It should not be performance sensitive, he said, because the error path should not be followed frequently. He wondered if any messages that were added would need to become part of the kernel ABI, thus be unchangeable, or if they could contain, say, a source file name and line number, which will obviously change. 

As an experiment, he tried redefining `EINVAL` and the other error codes as macros that would create a tracepoint with the source file and line number, but ran into multiple problems doing so. For example, those values are used in `switch` statements, conditional expressions, and in pre-setting a variable with an error code; in each case, a macro replacement will cause compilation or other problems. 

The alternative is to use different macros for the different kinds of uses of the error codes, so perhaps `ERR_TRACE()` in a return statement to place the tracepoint. More examples of the macros can be seen in his slides as displayed in the [YouTube video of the session](<https://www.youtube.com/watch?v=T1xVINgWL3w>). The problems with that kind of change are that it would have to be done manually, would add complexity, and would cause a lot of code churn. 

But Amir Goldstein thought that the code churn would be "localized to the person of interest" because it would only be done to a subsystem by its maintainer if the maintainer is interested in getting the extra information. Kent Overstreet said that the refactoring could be done [using Coccinelle](<https://docs.kernel.org/dev-tools/coccinelle.html>), rather than manually. He also thought there might be some overlap with the infrastructure recently merged for the [memory-allocation profiling work](<https://lwn.net/Articles/974380/>) that he has been doing; he uses a large number of error codes in bcachefs that effectively encode the source file and line number in them, though they are mapped to regular `errno` values before being returned to user space. 

Aleksa Sarai was not entirely sure how useful it would be to simply get the single data point of where the error code was set. In his debugging experience, seeing the entire call stack, as you can with ftrace, has normally been needed. 

Ted Ts'o said that it was not clear what the use cases were for this feature; was it meant to be for end users in some future RHEL kernel, for filesystem developers, or for some other use case entirely? There are different tradeoffs depending on the use cases, he said. As a user-space developer, Omar Sandoval said he would like to see an easy way to get a string that indicates where the `EINVAL` was generated, without having to deal with tracepoints or ftrace at all. 

Szeredi said that he sees the feature as being targeted at developers who are debugging these problems, possibly remotely. That is the use case that he has personally encountered most frequently. He can see that it might also be useful for returning information to applications. 

Christian Brauner said that he was probably responsible for adding 50 or more `EINVAL` returns to the kernel over the years; each time he does so, he wonders if he should also add a `pr_info()` call with some extra information. Traditionally, other kernel developers complain when an extra call with more information gets added, he said, but if he was working on his own project in user space, he would add them every time. They are seen as dmesg noise by some, however. 

Goldstein wondered how the kernel could provide the C library (libc) with access to the string containing the source file and line number. User-space programs access `errno` and use [`strerror()`](<https://man7.org/linux/man-pages/man3/strerror.3.html>) to get more information, so somehow the kernel needs to provide any extra information via a mechanism that libc can access. Szeredi had proposed putting a string in [`struct ptrace_syscall_info`](<https://elixir.bootlin.com/linux/v6.10.3/source/include/uapi/linux/ptrace.h#L82>), but libc does not have access to that. 

Overstreet said that the new error codes for bcachefs have "been enormously useful"; he tries not to reuse the codes at all so they effectively indicate the code location. David Howells said that the libc mechanism could be a new system call to retrieve the additional information; or user space could register a string buffer per thread with the kernel where that information could be stored. Ts'o returned to the use-case question, though; some users are only going to be interested in a high-level summary message, while developers may want a series of low-level failure messages. Those two use cases have different requirements and he was concerned that the discussion was getting complicated because it was trying to solve both at once. 

Speaking of complications, Howells noted that there can also be errors coming from a remote source, for example from a network filesystem. That seemed to bring the discussion to a close, though it is rather unclear what, if anything, had been decided. The session was toward the end of the third and final day of the summit, so attendees may well have been worn out at that point.
