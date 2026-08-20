---
title: Attaching programs to multiple tracepoints
url: https://lwn.net/Articles/1082948/
date: "July 22, 2026"
category: Tracing
author: "By Daroc Alden July 22, 2026 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
July 22, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Tracepoints in the kernel are useful for a variety of purposes: debugging, active monitoring, and performance measurements, among other things. Previously, any given BPF program could only be attached to a single tracepoint. Jiri Olsa has been working to change that, and led a discussion about his progress at the 2026 [ Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). That work has since been [ merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c49f336dbcf30ff8622d3725c54fe1c90e8ccd9c>), and can be expected as part of the 7.2 kernel. 

Other similar facilities in the kernel support multiple attachments; kprobes in particular support it, and can do most of the same things as tracepoints. The difference is one of performance, Olsa said. Tracepoints are slower to set up, but measurably faster to execute. His [ slides](<https://drive.google.com/file/d/132N1l5ndqMQwwOQKbyin38S_821C_3um/view>) included measurements of how many millions of executions per second each method could manage. Depending on exactly how the tracepoint and kprobe are attached, tracepoints can be about twice as fast. If tracepoints could be made to support multiple attachment without a regression in performance, they would be an attractive option for monitoring performance-sensitive operations. 

[ ![\[Jiri Olsa\]](https://static.lwn.net/images/2026/jiri-olsa-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1083260>)

Olsa then walked through a bit of the history of how tracepoints have been implemented. They use run-time code patching to redirect execution from a traced function to a generated trampoline that executes the attached hook before returning to the traced function. Originally, tracepoints were implemented using the old [ ftrace API](<https://lwn.net/Articles/365835/>), which has one-to-one correspondences between trampolines, functions, and [ ftrace objects](<https://elixir.bootlin.com/linux/v7.1.3/source/include/linux/ftrace.h#L436>). That is obviously not conducive to supporting multiple attachment. Menglong Dong [ tried a different approach](<https://lwn.net/ml/all/20250703121521.1874196-1-dongml2@chinatelecom.cn/>) that used a single, global trampoline for all tracepoints. That caused a performance regression because the trampoline needed to identify which function had been called, and the overhead of making that determination was too high. 

The new approach is to use a newer ftrace API that supports using a single ftrace object to configure multiple functions. Each function is still assigned its own trampoline, but the management of the trampolines can be handled as a group. Trampoline generation is fast, so regenerating several trampolines at once does not have a huge performance impact. 

One complication during development of the new multiple-attachment code was the handling of locking. Originally, each trampoline had its own lock that the code had to take in order to modify it. When many trampolines needed to be modified simultaneously, that could result in the code taking a large number of locks, bumping up against lockdep's 48-lock limit and causing problems for debugging. The solution was for Andrii Nakryiko to remove the locks from the trampolines and replace them with a pool of 32 locks that are shared depending on trampoline address; in practice, this allows almost as much concurrency when accessing trampolines, since the locks are only held briefly. 

Another problem — which was identified by the [ Sashiko patch-review system](<https://sashiko.dev/>) — is the interaction between tracepoints and live kernel patches. The latter also use the ftrace API in order to inform the ftrace subsystem when a live kernel patch has replaced a traced function. When such a function has a tracepoint attached to it, ftrace will notify the tracepoint code that the trampoline needs to be regenerated. Doing so requires taking the trampoline's assigned lock, but the thread might already hold locks (presumably in the ftrace subsystem, although Olsa did not specify) that are acquired after the trampoline's lock by other code, leading to a lock inversion — a deadlock. To remedy that, the code tries to acquire the lock in a loop, sleeping between iterations to temporarily release any mutexes that the thread holds. 

That isn't the bug, though: the problem is what happens when a live patch is removed. In that circumstance, the trampoline must be changed back, but the state of the locks is such that the code cannot attempt to acquire the trampoline lock in a loop. Therefore, if the trampoline lock is held when a live patch is removed, the trampoline may not be changed to refer to to the previous location of the traced function. Steven Rostedt suggested that this case should just return `EAGAIN` so that the live-patch-removal code knows to release locks and try again. Another developer thought that it was okay to ignore the problem, explaining that it was only a performance problem, not a correctness problem, because the only downside of not regenerating the trampoline in this case is an extra indirection from the location of the removed patch back to the original location of the function. 

The actual API to attach a BPF program to multiple tracepoints is relatively straightforward, taking a pointer to a program, a pattern specifying which tracepoints to attach to, and an optional set of [cookies](<https://grant.pizza/blog/bpf-cookies/>) to let the BPF program know which tracepoint it has been called from. 

```
struct bpf_link *
        bpf_program__attach_tracing_multi(
            const struct bpf_program *prog,
            const char *pattern,
            const struct bpf_tracing_multi_opts *opts);
                                         
        struct bpf_tracing_multi_opts {
            size_t sz;
            __u32 *ids;
            __u64 *cookies;
            size_t cnt;
            size_t :0;
        };
```

The API supports tracing function entries, exits, or both. It doesn't support altering the return values of functions or attaching to Linux security-module hooks. Nakryiko asked whether that was a limitation of the design, or if support for the other kinds of tracepoints could be added later. Olsa affirmed that the other kinds were possible to support, he just wanted to focus on the simplest cases first. He has not yet shared any patches to address the more complicated cases. 

Another limitation of this interface is what the BPF verifier can ensure, and therefore what is safe for attached BPF programs to do, Olsa explained. When attaching to a single tracepoint, the verifier can check the BPF program's access to that function's arguments, and therefore let the BPF program safely dereference any pointers to kernel objects that are passed to the function. When attaching to multiple tracepoints, the traced functions could have different signatures, so the verifier can't check the BPF program's accesses in a single pass. Therefore, a BPF program attached to multiple tracepoints is allowed to load the function arguments of the traced functions, but not to dereference any pointers it finds there. That is a limitation that will be difficult to lift without causing performance problems for the new API that obviate much of its benefit. 

One person asked how many functions Olsa expected a program to attach to at once, and whether this would cause performance problems by sending multiple inter-processor interrupts (IPIs) as the trampolines are created and attached. Olsa thought that most users of the new API would probably want to attach to hundreds of tracepoints at most — such as hooking into every system call implementation. On the other hand, that should not require hundreds of IPIs; the ftrace API uses some x86-specific tricks to avoid issuing multiple IPIs. That doesn't work on Arm, but support could be added relatively easily. 

Rostedt confirmed that he had written the code to handle that on x86, and that adding support for Arm would only require writing a slightly different kind of trampoline. Nakryiko asked whether there was a plan to do that work, to which Rostedt replied: ""If someone does it, I won't NAK it."" 

When pressed for details, Rostedt explained that Arm doesn't have a single instruction that pushes the current instruction pointer to the stack and then jumps directly to an address the way that x86's `call` does. The equivalent instruction on Arm, `bl` ("branch-and-link"), leaves the previous instruction pointer in a register. Therefore, patching a trampoline into a function requires altering more than one instruction, which makes an IPI necessary to ensure the CPU doesn't see an invalid intermediate state. 

If someone would just implement a trampoline that could be patched in using a single, atomic write in a way that respects Arm's rules for patching executable code, the same IPI-avoiding trick as x86 could be employed, and attaching to multiple tracepoints would become faster, Olsa said. By that point, however, the session had reached the end of its scheduled time, and discussion ended. At the time of writing, nobody has contributed such an Arm trampoline, although Olsa's patch set was merged on June 7.
