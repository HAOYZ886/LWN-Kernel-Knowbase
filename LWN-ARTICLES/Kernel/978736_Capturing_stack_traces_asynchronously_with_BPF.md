---
title: Capturing stack traces asynchronously with BPF
url: https://lwn.net/Articles/978736/
date: "June 19, 2024"
category: BPF
author: "By Daroc Alden June 19, 2024 LSFMM+BPF"
---

By **Daroc Alden**  
June 19, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Andrii Nakryiko led a session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) giving a look into the APIs for capturing stack traces using BPF, and how the APIs could be made more useful. BPF programs can capture the current stack trace of a running process, including the portion in the kernel during execution of a system call, which can be useful for diagnosing performance problems, among other things. But there are substantial problems with the existing API. 

The existing way to get stack traces in BPF is to create a BPF map for containing the elements of the trace, and then to call [ `bpf_get_stackid()`](<https://www.man7.org/linux/man-pages/man7/bpf-helpers.7.html>) from the BPF side, which returns a unique ID for the map, Nakryiko explained. Then in user space, the program could do a normal map lookup to retrieve the stack trace. The kernel also captures and stores some related information, such as the ELF build ID and the file offset, which helps identify what program the stack trace corresponds to for offline analysis. This API sounds fairly simple, but unfortunately it has a few quirks, he said. 

It can capture either a user-space stack trace or a kernel stack trace, but not both. The kernel supports capturing both, the BPF API just doesn't have a way to indicate that you want both, Nakryiko said. There is also no way to know how large the stack that was captured actually is; programmers just need to hope their stack-unwinding code stops at the correct point. These are both flaws, but they can't be too much of a problem, because nobody has complained. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The final quirk is that the kernel performs automatic stack deduplication. If the captured stack trace matches an existing one, the kernel returns the existing ID. This behavior sounds great in theory, since deduplication can save space, but the map used to store stack traces does not have any way to deal with hash collisions. Stack traces are hashed and placed into the corresponding slot in the map, but each slot can only hold one stack trace. Accordingly, hash collisions (where the hashes are the same, but the traces are not) are both frequent and unavoidable when capturing many stack traces. The API does let the programmer specify whether they wish to retain the old or new stack trace on collision, but that just leaves them with two bad options: lose data or corrupt data. The hash-based stack deduplication also makes clearing out entries from the map inherently racy. 

Nakryiko's proposed new API fixes these problems by letting the BPF program handle memory management: it provides a buffer, into which the kernel captures the stack trace, and then the BPF program is free to share that with user space in whatever way makes most sense for the application. All but one use case at Meta has switched to the new API, he noted. There are still some potential improvements to be made, however. Right now, stacks are captured synchronously. This is a problem, because the API can be called from anywhere, including a context where page faults are not permitted. 

If part of the program being inspected is paged out, that means the information stored on those pages could be left out of the capture. This only affects user-space stack traces, since the kernel always remains paged in. This is a particular problem for programs that use DWARF-based stack unwinding, since the DWARF debugging information is unlikely to be paged in when the capture is taken. 

Nakryiko would like the new API to be asynchronous, so that it can wait for necessary information (for user-space captures) to be paged in. That doesn't work for _kernel_ stack traces, however, because the kernel can't be paused to wait the way that user space can be. On the other hand, capturing a user-space stack trace can be postponed without changing the information returned by waiting until just before the kernel returns to user space, since the process is frozen until then. "Return to user space" is a nice context, he said, since the kernel can wait for memory to be paged in, etc. 

All of these separate constraints come together in the proposed API design. Nakryiko proposes having a function that returns a unique ID for a stack trace. The ID acts like a reservation — it is stable, and can be recorded, passed to user space, etc. Once the ID is received, a stack trace is captured into the reserved buffer. Kernel traces are captured synchronously, but user-space traces are captured on return to user space. Looking the stack capture up in the corresponding map returns `EAGAIN` until the capture is ready. The kernel doesn't perform deduplication, making deleting elements work in a sensible way. One audience member asked whether this meant that there could be identical stack traces with different IDs, and Nakryiko confirmed that this was the case. Daniel Borkmann noted that if the operations were asynchronous, it would also be possible to grow the map itself, which is not true of the existing API. 

Nakryiko did say that one part of the API was not yet thought out: how to let the user know when a stack trace is ready. The trivial solution, he said, was to not send a notification and force the users to poll. Slightly more complicated would be a map-wide epoll notification whenever any trace is ready. Alternatively, each slot in the map could have separate epoll notifications — but that would be a wasteful use of file descriptors. Finally, the most efficient approach would be to set up a BPF ring buffer into which IDs are put as they become ready, permitting efficient notification and consumption. 

One audience member pointed out that Nakryiko had pretty much just described the mechanism behind io_uring, and suggested that might be an appropriate mechanism. Nakryiko wasn't too sure about whether io_uring would be suitable, but acknowledged the possibility. Another audience member asked whether they could put a growable ring buffer in a [BPF arena](<https://lwn.net/Articles/961941/>). Nakryiko thought that would not be helpful, since there are already many ring buffers in the BPF subsystem, and reimplementing one in an arena wouldn't really help. He did note that if they switched to putting the whole stack trace into a ring buffer, they could potentially fit drop notifications in the ring buffer as well. 

Yu pointed out that the API could use a callback instead of a notification: run another BPF program on completion, and let the user decide how to notify user space. This prompted an extended discussion about different mechanisms, and how they traded off between flexibility and simplicity. Nakryiko did say that he was against a mechanism that tried to move too much functionality out of the kernel, since the kernel already has good built-in support for stack traces. One example is how user-space return probes can corrupt the stack trace — the kernel has all the necessary information to fix that up, whereas a more flexible mechanism would leave it up to the user. 

The discussion didn't reach a solid conclusion before the session time ended, but there are already many mechanisms in BPF and elsewhere in the kernel for notifying user space when an operation completes, so it seems unlikely for that to remain a sticking point of API design. BPF already has fairly good debugging support, given the tight integration with the BPF Type Format (BTF) and support for tracepoints in both the kernel and user space; it looks like some of the work in progress could help extend that support even more.
