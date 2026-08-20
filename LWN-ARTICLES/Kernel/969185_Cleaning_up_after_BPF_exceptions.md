---
title: Cleaning up after BPF exceptions
url: https://lwn.net/Articles/969185/
date: "April 15, 2024"
category: BPF
author: "By Daroc Alden April 15, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Daroc Alden**  
April 15, 2024

Kumar Kartikeya Dwivedi has been working to add support for exceptions to BPF since mid-2023. In July, Dwivedi posted [ the first patch set](<https://lwn.net/Articles/938435/>) in this effort, which adds support for basic stack unwinding. In February 2024, he posted [ the second patch set](<https://lwn.net/ml/bpf/20240201042109.1150490-1-memxor@gmail.com/>) aimed at letting the kernel release resources held by the BPF program when an exception occurs. This makes exceptions usable in many more contexts. 

BPF exceptions are somewhat dissimilar to exceptions in other languages. For one thing, they cannot be caught — any call to `bpf_throw()` will result in the BPF program exiting. There is a callback that the user can register to set the return code of the BPF program in the event of an exception, but there is no way for the program to recover. In the same vein, there are not different types of exceptions — all BPF exceptions behave the same way. BPF exceptions are subtly different from `exit()` because they do still unwind the stack. 

Currently, unwinding the stack doesn't make much difference. The BPF verifier prevents programs that hold resources (data structures, such as sockets, that must be disposed of in a specific way) from raising an exception by calling `bpf_throw()`. Letting them do so would be a problem because the current kernel is not prepared to release those resources, which would be a violation of BPF's safety guarantees. Dwivedi's new patch set takes advantage of the fact that BPF exceptions still unwind the stack to release the resources held by each function as its stack frame is unwound. For now, only some types of resource are supported — notably not including spinlocks or read-copy-update (RCU) structures — but future work can add additional types of resource over time. 

Or that's the theory, at least. As it stands, the BPF verifier does not always prevent programs from throwing exceptions with resources held. The [first patch](<https://lwn.net/ml/bpf/20240201042109.1150490-2-memxor@gmail.com/>) of Dwivedi's new set notes: 

> However, there currently exists a loophole in this restriction due to the way the verification procedure is structured. The verifier will first walk over the main subprog's instructions, but not descend into subprog calls to ones with global linkage. These global subprogs will then be independently verified instead. Therefore, in a situation where a global subprog ends up throwing an exception (either directly by calling bpf_throw, or indirectly by way of calling another subprog that does so), the verifier will fail to notice this fact and may permit throwing BPF exceptions with non-zero acquired references. 

That patch fixes the issue by making the verifier determine early on which functions can throw exceptions, so that information is available when performing the main analysis. The remaining patches in the set add a new pass to the verifier to let it collect the information needed to actually release any resources the program holds. 

In order to free resources the BPF program holds, the kernel needs to know two things: where they are, and what kind of resource they are. The new verifier pass walks the program and generates a map for each location through which an exception could cause the stack to unwind. Each one records which locations on the stack could hold releasable resources at that point in time. These maps also store the type of resource, which the verifier already has to track in order to ensure that resources are properly released in the course of normal execution. 

Not all relevant resources live on the stack, however. BPF has callee-saved registers (R6-R9) which might contain resources when `bpf_throw()` is called. To handle this, the patch set inserts a new hidden wrapper around `bpf_throw()` that explicitly spills those registers to the stack. 

At run time, when an exception is thrown, the kernel looks up the relevant stack map using the current instruction pointer. For each releasable resource, the kernel calls the release function associated with its type. Then the stack frame is unwound, and the kernel repeats the process with the stack map of the calling function. When unwinding the program, the kernel also needs to have the location of callee-saved registers recorded in the map, so that it can restore them to the correct location. This keeps the state of the program being unwound consistent, so that stack maps of earlier frames remain correct. 

One advantage of this approach is that subprograms that do not use exceptions don't incur any additional runtime overhead, because they do not need stack maps. In contrast, one complication is that it is perfectly legal for a BPF program to store different things in the same stack slot in different execution paths of the function, as long as the verifier can show that the types remain correct. A completely comprehensive approach to a stack map would therefore need to include some amount of runtime information about which execution path a function has taken. 

Dwivedi's patch set does not go that far. Luckily, it turns out that most real-world BPF programs do not actually use stack slots in this way. Dwivedi's patches "merge" stack maps from divergent execution paths when they have compatible types, and return an error when they do not. He investigated and found that existing BPF programs do not run into this error, and that merges of conflicting types were ""unlikely to often occur in practice"". 

There is one special case, however. It is somewhat common for a program to acquire a resource conditionally, which means that its stack slot might contain a null pointer. The new verifier pass handles this by merging other types with null pointers when necessary. In the end, it requires that all the execution paths of a function either store the same type of resource in the same slot, or leave a null pointer there. This allows the verifier to coalesce all of the maps for a given function, preventing a potential combinatorial explosion. 

Eduard Zingerman raised [ some concerns](<https://lwn.net/ml/bpf/94ee37372c90c28980246ab803dffb3d2b63be35.camel@gmail.com/>) with that approach, saying that he worried about the ""possibility that frame merge might fail because of some e.g. clang version changes"" that modify how the compiler chooses to lay out the stack. Zingerman suggested a run-time approach that tracks resources as they are acquired and released by the actual program instead, saying such an approach ""seems simpler than frame descriptors tracking and merging"", and that it could support aborting BPF programs at any point, instead of only at calls to `bpf_throw()`. The downside would be run-time overhead, even for programs that never actually throw an exception. 

Dwivedi [ responded](<https://lwn.net/ml/bpf/CAP01T74kov3JKJFvpaqz6CjEkzBMbOfDLc6Xjg_n5=g9osApjA@mail.gmail.com/>): ""I went over this option early on but rejected it due to the typical downsides you observed"". He went on to explain the overhead such a runtime approach would require in detail, concluding by saying: ""I just think this is all unnecessary especially when most of the time the exception is not going to be thrown anyway, so it's all cost incurred for a case that will practically never happen in a correct program."" 

David Vernet also [ reviewed](<https://lwn.net/ml/bpf/20240212193547.GB2200361@maniforge.lan/>) the patch set, pointing out that the new pass looks fairly similar to the existing `check_max_stack_depth_subprog()` code, and asking whether they could be combined. Dwivedi [ agreed](<https://lwn.net/ml/bpf/CAP01T76dLSoEuaOSe0NGm+ainmJF1XNBpJazY0w+aB3R0KdMbw@mail.gmail.com/>) that this would be a good idea. He plans to incorporate that work (and a related refactoring of the stack-depth-checking code) into version two of the patch set.
