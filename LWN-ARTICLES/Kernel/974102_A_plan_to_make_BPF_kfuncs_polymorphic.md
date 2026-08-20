---
title: A plan to make BPF kfuncs polymorphic
url: https://lwn.net/Articles/974102/
date: "May 20, 2024"
category: "BPF-kfuncs"
author: "By Daroc Alden May 20, 2024 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
May 20, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

David Vernet kicked off the BPF track at 2024's BPF track at the [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) with a talk about polymorphic kfuncs — or, with less jargon, kernel functions that can be called from BPF which use different implementations depending on context. He explained how this would be useful to the [sched_ext](<https://lwn.net/Articles/972710/>) BPF scheduling framework, but expected it to be helpful in other areas as well. 

Alexei Starovoitov gave a talk later in the conference about the history of BPF, including [ the origin](<https://lwn.net/Articles/856005/>) and motivation for kfuncs — stay tuned for an article on that. For now, knowing more about kfuncs is not really needed to understand Vernet's problem and proposed solution. 

There are 151 kfuncs in the kernel as of version 6.9, so it should probably not be too surprising that they vary wildly. Some kfuncs, Vernet pointed out, are used for extremely common, basic functionality — such as the functions for acquiring and releasing locks. These kfuncs have the same meaning and implementation in every possible context, because what they do is fairly simple. Other kfuncs, however, can have context-specific semantics. Some may only have "any meaning at all [...] within specific contexts", Vernet said. 

[![\[David Vernet\]](https://static.lwn.net/images/conf/2024/lsfmm/david_vernet-small.png)](<https://lwn.net/Articles/974117>)

One example of this is the functions for manipulating dispatch queues — structures used in sched_ext to store lists of pending tasks. Vernet called them the basic building blocks of scheduler policy. One of the main functions for manipulating them from BPF, `scx_bpf_dispatch()`, always has the same meaning: adding a task to a different queue. But when called from different BPF callbacks, there are subtle variations in how the function can be used. 

When called from a `select_cpu()` or `enqueue()` callback, `scx_bpf_dispatch()` cannot drop the run-queue lock relevant to the task, and can only dispatch tasks to the CPU that triggered the call. Furthermore, only tasks that are being woken or enqueued can be dispatched. 

In contrast, when called from a `dispatch()` callback, `scx_bpf_dispatch()` is free to drop the run-queue lock, dispatch to any CPU, and dispatch multiple tasks. The difference is that `dispatch()` is called by a CPU that is about to otherwise go idle, and so there is no existing work on the CPU that needs to be carefully worked around. 

In both cases, `scx_bpf_dispatch()` presents the same logical API, but the differing constraints mean that the implementation in these two cases is quite different. Right now, the code tracks which case it is in with a per-CPU variable, and then uses that to choose which implementation to use. "So you can work around it," Vernet admitted, but he wanted to see if the implementation could be better. 

#### Vernet's proposal

Right now, every kfunc is associated with a [ BPF Type Format](<https://docs.kernel.org/bpf/btf.html>) (BTF) ID. This is an ID used to represent the kfunc in the debugging information for the BPF program, but it is also used with the BPF instruction that calls a kfunc to indicate which one it wants to invoke. When the BPF program is loaded and then just-in-time compiled, the BTF IDs get resolved, and the resulting code can call them directly. 

Vernet suggests extending this mechanism by having the BPF verifier support multiple kfuncs with the same ID — whenever it encounters a call to a kfunc, it would ask the subsystem associated with that kfunc ID what the real kfunc should be (using a new callback). The subsystem would then reply with a "concrete" kfunc ID, and loading would proceed in the same way. This approach moves the tracking of the context of a call from run-time to load-time, and eliminates the need for tracking the state in a per-CPU variable. 

Vernet said that the advantage of this approach is the ergonomic API it presents, and the control it gives subsystems over how their kfuncs can be called. But the approach does have its drawbacks. For one thing, adding additional callbacks in the verifier threatens to make one of the most complicated parts of BPF even more so. For another, it would use load-time logic for what is really a static configuration — if the compiler understood the different contexts that the kfuncs care about, the correct kfunc implementation could be chosen at build-time. 

A build-time configuration would be nicer, Vernet stated, but it would be "kind of a pain in the neck to implement". He suggested that implementing it statically was probably not a high priority. Vernet did think any mechanism for polymorphic kfuncs would probably be useful to areas of the kernel other than sched_ext. 

#### Discussion

The other attendees had questions about Vernet's proposal. One member of the audience pointed out that there is already a similar mechanism for BPF helper functions (a _different_ kind of kernel function callable from BPF programs, with a different interface), and asked that Vernet "look at this more holistically". Vernet replied that the equivalent aspect of helper functions lets the implementations differ depending on the BPF program type — so the same helper function can be implemented differently for a BPF program attached to a trace point or registered as a callback. But that approach won't work for his use case, because the program types in question are not sufficiently granular. As far as the verifier is concerned, all of the callbacks involved in sched_ext are of the same type, because they are all `struct_ops` programs (a mechanism where different parts of the kernel can define a struct full of function pointers to which BPF programs can be attached). He wants to be able to handle calls from different `struct_ops` programs differently — which almost certainly requires information the verifier doesn't have, since it is the other subsystems or modules which define `struct_ops` callbacks that would know which functions should be handled differently. 

The discussion went back and forth a little bit, with the other attendee trying to identify ways that the mechanism could be generalized beyond `struct_ops` programs. Vernet agreed that "if we can abstract it that would be much better for sure," but didn't seem to think that the existing helper mechanism was a suitable basis for that. 

Another member of the audience asked whether it would be possible to have kfuncs that behave differently based on the type of their arguments. The motivating use case would be to enable different data types being inserted into a BPF map to be handled differently. "I want to skip the ownership check when the argument is an `sk_buf`", they explained. Vernet agreed that this would be technically feasible, since the verifier knows the types of the arguments to the kfunc. The question, in Vernet's eyes, is whether this mechanism would be confusing. 

The first participant in the conversation suggested that this use case could be served just by adding new kfuncs and letting the developer use the right one. The second commenter pushed back, saying that they did not want to introduce many new kfuncs for what is effectively the same behavior — especially not when it seems likely that there will be more types that need special handling to keep in maps in the future. Vernet agreed that it makes sense to give kfuncs the flexibility to decide what they want to do. 

That was the end of the discussion at the time, so it remains to be seen whether the proposal will be adopted, and if so in what form.
