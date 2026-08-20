---
title: "An instruction-level BPF memory model"
url: https://lwn.net/Articles/976071/
date: "June 4, 2024"
category: BPF
author: "By Daroc Alden June 4, 2024 LSFMM+BPF"
---

By **Daroc Alden**  
June 4, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

There are few topics as arcane as memory models, so it was a pleasant surprise when the double-length session on the BPF memory model at the [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) turned out to be understandable. Paul McKenney led the session, although he was clear that the work he was presenting was also due to Puranjay Mohan, who unfortunately could not attend the summit. BPF does not actually have a formalized memory model yet; instead it has relied on a history of talks like this one and a general informal understanding. Unfortunately, ignoring memory models does not make them go away, and this has already caused at least one BPF-related bug on weakly-ordered architectures. Figuring out what a formal memory model for BPF should define was the focus of McKenney's talk. 

LWN has written about [memory models for kernel code](<https://lwn.net/Articles/967049/>) before. Briefly, accessing memory is so slow that modern CPUs put a great deal of effort into minimizing and parallelizing requests to memory. This often takes the form of caching, but can also involve reordering memory accesses. This means that, from the point of view of a thread running on another CPU, memory operations can appear to occur out of order. A memory model says which reorderings are permitted, and which instructions programmers can use to control this behavior. "Strong" memory models are those that permit fewer reorderings (and therefore often require less explicit synchronization), while "weak" memory models are more permissive. 

McKenney opened the session by noting that — in an example of weak ordering typical of memory-modeling work — the patches he hoped to result from the talk had actually already been sent to the mailing list. Mohan had done an impressive amount of work in the week leading up to the summit, including [ fixing several problems](<https://lwn.net/ml/linux-kernel/20240505123340.38495-1-puranjay@kernel.org/>) with atomic instructions in BPF on weakly-ordered architectures and [ adding code](<https://github.com/herd/herdtools7/pull/855>) to [herd7](<https://github.com/herd/herdtools7>), a memory model simulator, in order to show the changes were correct. 

After acknowledging Mohan's work, McKenney then walked the attendees through the common assumptions embedded in BPF code about memory ordering, which amount to an informal memory model. BPF has a set of atomic instructions — notably `xchg` and `cmpxchg` — that provide full ordering. All CPUs and tasks agree that the side effects of all instructions that come before an atomic instruction are visible before it, and all effects of subsequent instructions are not visible until it has been executed. McKenney noted that this was "straightforward, but really important". Other atomic instructions, such as atomic adds, are unordered, unless they specify the optional `FETCH` flag. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The other source of memory ordering guarantees in BPF is jump instructions. Unconditional jumps don't affect ordering, but conditional branches do under some circumstances. When there is a load instruction, the result of which is used in the comparison for a branch, and then that value is stored following the branch, the store is guaranteed to occur after the load. McKenney noted that this was a pretty subtle point, and means that optimizing BPF programs requires some care. Alexei Starovoitov interjected: "Don't run BPF assembly through an optimizing compiler", which produced some chuckles. 

Those properties form the bare minimum of what most BPF programs currently assume about ordering. But in the absence of a formalized model, it's impossible to say whether any given bug resulting from memory operations being reordered is a bug in the BPF program, the compiler, or the BPF just-in-time compiler (JIT). McKenney has four main goals for a formal memory model: it should operate at the level of individual instructions, so that it is directly applicable to the JIT; it should be consistent with the [ Linux kernel memory model](<https://lwn.net/Articles/718628/>) (LKMM); it should support low-overhead mappings to supported hardware; and it should be able to grow as BPF does. 

[ ![\[Paul McKenney\]](https://static.lwn.net/images/2024/paul-mckenney-small.png) ](<https://lwn.net/Articles/976313>)

These goals pose some challenges. In order to be efficiently mapped to different kinds of hardware, the memory model should avoid forbidding reorderings that those architectures permit — which necessarily means that it will end up being a "weaker" memory model than any of those architectures alone. The benefit is that, if the formal model is successful, the JIT should not need to emit synchronization instructions or memory barriers in most cases. 

McKenney thought that some people might wonder: don't we already have the Linux kernel memory model? Why not just map BPF assembly to LKMM C? Unfortunately, that doesn't really work, he said, at least not trivially. High-level languages like C and low-level languages like assembly have different event structures that make a simple mapping impossible. Assembly also has additional constraints around individual registers or similar low-level constructs that the LKMM doesn't address. He did think starting from an existing memory model was a good idea, however, since the basics of any memory model are likely to be fairly similar. 

Of the existing memory models, McKenney considered a few alternatives. X86 is much too strong, and PowerPC is not actively developed (and missing atomic instructions). Ultimately, he chose ArmV8 as a possible starting point, since it is actively maintained and full-featured. The downside is that it includes some irrelevant hardware features, and that it is still stronger than PowerPC, so some changes will be needed. 

McKenney showed the attendees a few selected sections of the ArmV8 memory model specification, to give a feeling for what adapting it would be like. Then he went into a series of examples of things "that kinda hit me over the head really hard" — unintuitive consequences of the model that would need to be considered while adapting it to BPF. 

#### Examples

The first example dealt with dependencies between loads and stores. Suppose that there are two reads (called R1 and R2), and one write (called W1). In the program, the instructions occur in the order R1, R2, W1. When executed, R1 reads from some known address (a pointer), returning a second address that R2 reads from. W1 writes to a different address. The question is: is the CPU allowed to re-order the write to occur before the reads? After describing the scenario, McKenney paused to give the experienced kernel developers in the room a chance to guess. 

As it turns out, both ArmV8 and PowerPC forbid that reordering, but the LKMM _doesn't_. The reason has to do with aliasing — the CPU, when it is executing those instructions, can't know whether R2 and W1 will access the same location until read R1 actually resolves. So of course W1 can't occur before R1, because that might cause R2 to read the wrong value. The LKMM, however, operates on a more abstract level. If the pointers involved in this scenario are passed in as arguments to a function written in C, the LKMM is allowed to assume that they don't alias. Since they don't refer to the same location, reordering W1 before R1 doesn't cause any problems. 

In this case, BPF is in a position more like a CPU than like the C abstract machine. The JIT could theoretically know whether two accesses alias, but that would seriously complicate code generation, so it makes more sense to adopt the same restriction as ArmV8 and forbid the reordering. 

The second example was somewhat more esoteric. Suppose that you have two threads and one variable X. One thread reads X twice, in read operations R1 and R2. The other thread writes to X, with an operation W1. Suppose that R2 occurs before W1 (i.e., it doesn't see the value written). All the operations use the platform's equivalent of [ `READ_ONCE()` and `WRITE_ONCE()`](<https://lwn.net/Articles/846700>), i.e. atomic reads and writes with weak ordering guarantees. ~~Does R1 also necessarily occur before W1?~~ [McKenney [clarified in the comments](<https://lwn.net/Articles/977385/>) that this is not quite the correct framing. The proper question is: does R2 being unable to read a value older than the value read by R1 have any other ordering consequences?] After a moment of silence, one audience member pointed out that the answer can't possibly be "Yes", because otherwise McKenney wouldn't be asking, producing another round of laughter. 

And indeed, on PowerPC, ~~operation R1 is allowed to read the value written by W1, even though R2 doesn't.~~ even though R2 might read a value later than that of R1, that doesn't impose additional ordering consequences. To prove it, McKenney showed the scenario running in herd7, showing that it found a proof that the reordering is possible. The scenario did need to include a few additional complications in order to actually make the test meaningful, and McKenney spent a while walking through the requirements to actually measure the scenario so that the audience could understand what was happening in the example. "If you're serious about understanding this — which I'm not sure you should be —", McKenney advised, "you can consult the example in [ the book](<https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.2011.01.02a.pdf>)", referring to his book about parallel programming. 

McKenney noted that this is an example of PowerPC being exceedingly weak — ArmV8 forbids the same reordering. The BPF developers probably prefer to go with PowerPC's version, however, so that they can avoid emitting extra memory synchronization instructions during code generation. BPF doesn't currently have an equivalent of `READ_ONCE()` and `WRITE_ONCE()`, so the example doesn't necessarily apply, but it's something to remain mindful of. 

In short, even starting from the ArmV8 specification will require a certain amount of adaptation and careful thought, McKenney summarized. He then showed some of the support for BPF code in herd7. Mohan is working to extend that support, which will probably prove useful for validating any proposed formal model. The session ended with some discussion about where the memory model should live once it was written. McKenney plans to get it into the kernel's `tools/standardization` or documentation directories, but isn't particular about where exactly it ends up.
