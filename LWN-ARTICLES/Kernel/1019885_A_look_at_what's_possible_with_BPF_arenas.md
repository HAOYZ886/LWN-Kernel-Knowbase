---
title: "A look at what's possible with BPF arenas"
url: https://lwn.net/Articles/1019885/
date: "May 13, 2025"
category: "BPF-Memory management"
author: "By Daroc Alden May 13, 2025 LSFMM+BPF"
---

By **Daroc Alden**  
May 13, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

[ BPF arenas](<https://lwn.net/Articles/961941/>) are areas of memory where the verifier can safely relax its checking of pointers, allowing programmers to write arbitrary data structures in BPF. Emil Tsalapatis reported on how his team has used arenas in writing [ sched_ext schedulers](<https://lwn.net/Articles/922405/>) at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. His biggest complaint was about the fact that kernel pointers can't be stored in BPF arenas — something that the BPF developers hope to address, although there are some implementation problems that must be sorted out first. 

Tsalapatis started by saying that he and his team have been happy overall with arenas. They have used arenas in several different scheduler experiments, which is how they've accumulated enough feedback to dedicate a session to. In particular, with a few tweaks, he believes that arenas could be useful for allowing the composition of different BPF schedulers. 

[ ![\[Emil Tsalapatis\]](https://static.lwn.net/images/2025/emil-tsalapatis-lsfsmmbpf-small.png) ](<https://lwn.net/Articles/1020264>)

Sched_ext is the kernel's framework for writing scheduling policies in BPF. The mechanism is designed to allow scheduler developers to rapidly experiment with alternative approaches, but it has also seen some success in allowing a user-space control plane to communicate important information about processes to the kernel. 

In the BPF schedulers Tsalapatis has worked on, the main way they represent scheduling state is with C structures shared between user space and the kernel. These structures have statistics, control info, CPU masks, and anything else necessary to make scheduling decisions about a task. Often, that includes identifiers that refer to other parts of the scheduler, such as affinity layers (sets of tasks with similar scheduling needs and characteristics) or related tasks. None of that is ""really possible with map-based storage"". 

Sharing the same information using BPF maps, he explained, would involve a lot of pointer-chasing between different maps. BPF maps aren't well suited to representing complex data structures. BPF arenas, on the other hand, are essentially a ""big blob of memory"" that the programmer can do whatever they would like with. That freedom let Tsalapatis's scheduling code be more expressive, and in turn enabled him to implement features that would have previously been infeasible, such as faster migrations within the last-level cache domain. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Arenas have one big drawback, however: they only store data. Even if, from the point of view of the CPU, there's no real difference between a pointer in the kernel and a pointer in user space, there's a big difference from BPF's point of view. The verifier prevents BPF programs from creating pointers to kernel objects, for security reasons. Even if a program writes a kernel pointer to an arena, the verifier doesn't allow it to be used once it is read back. 

So, in practice, sched_ext developers break their data structures into two parts: one part in the arena, for ease of use, and one part stored in a BPF map, to hold references to kernel objects. This works, but it's inconvenient, and obviates many of the advantages of arenas. Tsalapatis and his team partially worked around the problem by creating their own spinlocks backed by the arena, so that they didn't need to manage pointers to kernel spinlocks. That only covers the data structures that they can easily replicate in BPF, however. 

Tsalapatis's main question for the BPF developers was: is it possible to allow kernel pointers to be stored in an arena? What needs to be figured out to make that happen? 

He also had a handful of related usability concerns. Right now, when they need to pass data from their arena-backed structures to a kernel function, they have to copy it to the BPF stack, which Tsalapatis described as ""kind of clunky"". Worse, helper functions can't be written in a way that is indifferent to whether a structure is stored in an arena or on the stack — the verifier treats the types differently. This is because the lower bits of arena pointers are the same between kernel space and user space, to facilitate the writing of shared data structures. In turn, that imposes special requirements on a BPF program when it accesses data through an arena pointer. So the helper functions either need to be duplicated or inlined, neither of which is ideal. 

Alexei Starovoitov was grateful for the feedback on arenas, but said that he had ""good and bad news"" about Tsalapatis's request. Work is already in progress on many of the things he asked for. Unfortunately, the work is still in progress because there are difficult problems to solve, so any fixes are going to take a long time, Starovoitov said. In particular, it's not clear how to let the verifier mix stack-backed and arena-backed pointers in a reasonable way. 

Tsalapatis said that using inline functions (which let the verifier base the type information on the specific call site in question) _does_ work, it is just less convenient than it could be. Starovoitov agreed that it would be better to fully support mixed stack/arena pointers, but simply didn't have any idea how to make it work at present. The BPF just-in-time (JIT) compiler uses the type information that the verifier attaches to pointers in order to emit correct loads and stores, for one thing. 

Starovoitov was curious what kinds of kernel pointers Tsalapatis wanted to store in arenas; thus far, he had mentioned CPU masks and spinlocks, but Starovoitov expected a scheduler to need more than that. Tsalapatis explained that for other data structures, such as the representation of a task, the BPF scheduler often needs its own version anyway, so it's less inconvenient to store an index into a map for the kernel version. 

With the problem somewhat defined, the BPF developers launched into an extended discussion about how best to permit storing kernel pointers in arenas. Options mentioned included adding an extra level of indirection (essentially automating the technique of storing kernel pointers in a map and referencing them by index) and adding a ""shadow page"" to contain kernel pointers. 

Depending on how exactly it is implemented, that approach could have substantial implications for the memory use of BPF arenas. One proposed approach was to allocate a separate page, not accessible by user space, for each page of the arena in which kernel pointers are stored. That could result in a situation where reading from an address in an arena returns different data depending on whether the program expects to read a kernel pointer or plain data, which was somewhat unpopular. Another variant of the idea would be to allocate a packed bitmap tracking the validity of stored kernel pointers, a bit like [ CHERI](<https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/>) hardware does. That has its own problems, however, and complicates every access to the arena. Ultimately, the discussion did not come to a conclusion. 

Tsalapatis's last piece of feedback on arenas was not directly related to his earlier questions about storing kernel pointers. One of the key advantages of arenas is that the verifier doesn't need to validate operations inside an arena. This makes them flexible, but it can also make it more difficult to track down bugs. Like in a typical C program, and unlike in other kinds of BPF programs, accesses past the end of a structure don't cause an immediate verification failure. Tsalapatis asked everyone to think about ways to help combat this, while still making arenas useful. 

Starovoitov explained that several in-progress BPF features would help with that, too. For example, Kumar Kartikeya Dwivedi has been working on [ adding a standard error output stream](<https://lwn.net/ml/bpf/20250414161443.1146103-1-memxor@gmail.com/>) for BPF programs that could be used to report problems like page faults in an arena. The ongoing work to [ allow the cancellation of BPF programs](<https://lwn.net/Articles/1010404/>) is another thing that could make handling run-time errors in BPF programs more feasible. 

BPF arenas are fairly recent; they were introduced in kernel version 6.9. Based on the discussion around Tsalapatis's feedback, it seems like they have not yet achieved their final form, and could end up being a complete alternative to BPF maps, if the kernel developers can overcome some last few hurdles.
