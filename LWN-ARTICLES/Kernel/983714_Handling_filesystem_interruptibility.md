---
title: Handling filesystem interruptibility
url: https://lwn.net/Articles/983714/
date: "August 5, 2024"
category: Filesystems
author: "By Jake Edge August 5, 2024 LSFMM+BPF"
---

By **Jake Edge**  
August 5, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

David Howells wanted to discuss changing the way filesystem code handles the ability to interrupt or kill operations, in order to fix some longstanding problems with network (and other) filesystems, in a session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). As noted in his [session proposal](<https://lwn.net/ml/all/2701318.1706863882%40warthog.procyon.org.uk/>), some filesystems may be expecting to not be interruptible, but are calling code that can take locks and mutexes that are interruptible (or killable), which are effectively changing the state of the task incorrectly. He would like to find a solution for that problem. 

The interruptibility here refers to signal handling. An interruptible process will respond to any signals that are not masked or ignored. Killable is a variant of interruptible that will only respond to fatal signals. 

There are multiple places with locks and such that could be taken using the `*_interruptible()` and `*_killable()` variants, but those override the higher-level non-interruptible setting. Some kind of mass change is not really practical to address the problem, Howells said, so it will need to be done incrementally. He proposed a multi-year effort to switch to explicit begin and end functions to bracket non-interruptible regions, in a way that is analogous to how hardware interrupts are disabled. Code could disable interruptibility, which would be tracked with a counter, then reenable it when the critical section is finished. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

For example, an overlayfs filesystem might include a network filesystem as one of its layers. The overlayfs might not take interruptible locks, but the network filesystem might do so, which results in operations that get interrupted in a way that overlayfs does not expect. Ted Ts'o thought that a change like what was described might be useful in some contexts, but did not think that "it would be something we would want to use all over the place". The interruptible status of a particular mutex, for example, is local to the code that takes it. Unlike the [effort to switch to `GFP_NOFS`](<https://lwn.net/Articles/976355/>), where the eventual plan is to convert everything to use it, this change would only be needed for specific calls. 

Both Kent Overstreet and Dave Chinner asked for more concrete examples of the problem being solved and how the code would need to change to accommodate Howells's proposal. The biggest problem he has encountered, Howells said, is that `sendmsg()` is interruptible, but that an NFS filesystem might be mounted as non-interruptible. "NFS thinks it is not interruptible, but it is because it is using the network interfaces that are." He noted that a conversion would eventually mean that many of the interruptible (and killable) variants of lock and mutex functions could be removed. 

Chinner and others objected to that, saying that there will still be a need for those variants. There were also various objections because many of the calls to `mutex_lock_interruptible()` are not checked for an error return, though there were multiple people all talking at once making it somewhat hard to follow. Al Viro was also concerned about deadlocks resulting from the changes proposed. 

Viro said that handling signals (such as from someone using control-C) is the responsibility of the caller of the network function; an NFS mount with `-o hard` does not want or expect its operations to be interrupted, though, Howells said. However, calling `mutex_lock_interruptible()` is only applying the interruptibility to that specific call, Wedson Almeida Filho and Ts'o said, not to the whole region between it and the unlock call. Ts'o said that without a specific patch changing a particular code path where there is a problem, it will be difficult for attendees to determine whether it makes sense or not; meanwhile, he reiterated that he did not see a justification for a widespread change. 

Instead of having a call to bracket the regions of non-interruptibility, Viro asked, why not just disable signals for the region? But Howells said that `SIGKILL` cannot be masked, though Christian Brauner pointed out that the kernel can mask that signal even though user space cannot. 

Jan Kara agreed with the overall approach, saying that there is a real problem for callers who do not expect to get interrupted. But Brauner was concerned about how someone looking at `sendmsg()`, which is clearly interruptible, would be able to recognize that in some contexts it can be called in such a way that it is not interruptible. Howells acknowledged that could be a problem. 

Chinner suggested having a variant of `sendmsg()` that is not interruptible, but Howells said that there are multiple calls like `sendmsg()` that are affected. "The documentation of the uninterruptible state is completely decoupled from where we need to apply that state", Chinner said. It would require large comments wherever these functions are being called, describing how that can happen and what code paths are affected. "Otherwise it is unmaintainable." 

Viro said that it sounds to him like what Howells wants is to be able to suspend signal delivery in the network code at times. The `TASK_INTERRUPTIBLE` and `TASK_UNINTERRUPTIBLE` states are for sleeping processes, Viro continued, which get changed when the task gets woken up, but the state that is really desired "smells like 'I want signal delivery suspended'" until the end of the code region. 

Ts'o agreed, noting that the change could be done without adding new infrastructure and a task flag. Howells said that manipulating the signal mask would affect other threads in the process, though, which Ts'o acknowledged as a problem. Viro said that an alternative might be to simply skip the thread in question when doing signal delivery; "basically it is a 'don't bother me'". 

The session ran out of time as that was being discussed, but the picture that emerged is that patches are needed to focus the discussion. As of yet, there is no video for this session in the [2024 LSFMM+BPF playlist at YouTube](<https://www.youtube.com/playlist?list=PLbzoR-pLrL6oj1rVTXLnV7cOuetvjKn9q>).
