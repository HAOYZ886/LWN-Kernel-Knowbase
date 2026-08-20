---
title: Indirect calls in BPF
url: https://lwn.net/Articles/1017439/
date: "April 21, 2025"
category: BPF
author: "By Daroc Alden April 21, 2025 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Daroc Alden**  
April 21, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Anton Protopopov kicked off the BPF track on the second day of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit with a discussion about permitting indirect calls in BPF. He also spoke about his continuing work on [ static keys](<https://lwn.net/Articles/977993/>), a topic which is related because the implementation of indirect jumps and static keys in the verifier use some of the same mechanisms for tracking indirect control-flow. Although some design work remains to be done, it may soon be possible to make indirect calls in BPF without any extra work compared to normal C. 

[ ![\[Anton Protopopov\]](https://static.lwn.net/images/2025/anton-protopopov-lfsmm-small.png) ](<https://lwn.net/Articles/1017548>)

BPF has actually had preliminary support for indirect calls for a long time, Protopopov said. LLVM's BPF backend has supported emitting indirect-jump instructions since 2017\. Tools such as libbpf, objdump, and bpftool are also all set up to handle indirect calls correctly. The only obstacle to using indirect function calls was support for them in the verifier — but he was able to fix that with a reasonably compact patch. He didn't share the whole patch, because it is not yet complete, but at a minimum it depends on [his existing patch set](<https://lwn.net/ml/all/20250318143318.656785-1-aspsk%40isovalent.com/>) that introduces a new type of BPF map called an instruction set. 

Supporting indirect calls lets users of BPF write some programs in a more straightforward way. Protopopov shared the example of a reverse-polish-notation calculator that used function pointers to represent operations. That demonstrated that his change allows the verifier to understand function pointers assigned to a single variable, but there's still more work to be done. 

The next step is to allow BPF programs to fill a table with function addresses, index into that table, and then call the resulting function pointer. Among other things, that will allow the implementation of efficient bytecode interpreters in BPF, which are currently not really feasible. The problem is that the simple, self-contained change to the verifier that lets it handle indirect calls needs more supporting changes to work for arrays. Protopopov plans to use instruction-set maps to represent tables of function pointers. 

Instruction-set maps are essentially a table of addresses referencing a specific type of instruction in a BPF program. They are initially filled by user space, but become read-only when the BPF program is loaded. The verifier keeps the instruction addresses up to date as it performs dead-code elimination and other transformations — hence the need for a special map, as opposed to just letting BPF code use hard-coded offsets. The layout of BPF code can shift substantially during loading. When an instruction-set map is used to represent a table of function pointers, the verifier will verify each function individually with the same set of starting assumptions, so it will be safe to call any of the functions in the table interchangeably. 

Protopopov asked for feedback on a few details of the implementation, but the audience didn't see any immediate problems with his plan. He also asked for some examples of more real-world use cases for indirect function calls, so that he could ensure his solution would work for other users as well. 

James Bottomley asked about the possibility of passing function pointers as function parameters to act as a callback. Protopopov said that there is already a kfunc (kernel function callable from BPF) that can be used to invoke a callback from one BPF function to another, so the verifier can already handle that case in theory. Bottomley agreed, but thought that being able to pass function pointers — as is done throughout the kernel — was more ergonomic. Protopopov thought that passing function pointers as function parameters should work with his changes. 

Alexei Starovoitov suggested that Protopopov should add a test based on having a large `switch` statement, which modern compilers will turn into an indirect jump through a table when there are enough cases. If that works smoothly, then users should be able to just write `switch` statements, without caring about whether they turn into indirect jumps or not. Protopopov said that he preferred testing indirect calls to indirect jumps, but did agree that they all ultimately depended on the same implementation. 

#### Static keys

When that topic didn't take the full time slot for his talk, Protopopov provided an update on the in-progress support for static keys in BPF programs. Static keys are a mechanism in the kernel that use run-time code patching to implement low-cost configurable branches. Protopopov would like to have the same kind of efficient run-time configuration in BPF programs — but self-modifying code is notoriously hard for static analysis to handle. 

His implementation of static keys uses instruction-set maps as well, to track which places in the code are allowed to be modified. One key difference from their use for indirect calls is how those instruction-set maps are made available to the program. With indirect calls and jumps, the BPF program's instructions directly reference the appropriate map. With static keys, the `may_goto` instructions pointed to by the map (for the static key machinery to update at run time) don't actually reference the map. So in that case, the instruction-set map needs to be passed into the program using a file descriptor, so that it can be referred to later. 

One member of the audience had questions about how that could work: BPF maps that are defined in compiled BPF objects are shared across the different instances of the program attached to different points in the kernel. How, then, could a single map be kept up to date, when the verifier could change different instances of the BPF program in different ways? That is the biggest question about all of this, Protopopov responded. 

With the code in its current state, attempting to create a second instance of a BPF program that uses an instruction-set map simply fails — the map is already frozen when the verifier goes to modify instruction locations, so the verification process fails. A possible solution could be to create multiple maps: one for each instance of the program that will be attached to the kernel. The downside of that approach is that the existing method for updating BPF static keys will need some work to cope with having multiple maps. 

Protopopov then asked the rest of the assembled BPF developers whether they had any suggestions for a more elegant way to handle the problem. This prompted a flurry of ideas, including reworking the static-key machinery to not rely on instruction-set maps, making instruction-set maps natively handle multiple programs, and a few others suggested in some rather hard to follow discussion. 

Ultimately, Starovoitov was of the opinion that Protopopov should start by getting a simple `switch` statement that compiles to a jump table working, and then see about adapting things from there. Protopopov wasn't sure that was a good plan, because it doesn't solve the problem of static keys referencing multiple maps, but Starovoitov advised him to try adding an additional layer of indirection, to allow the static key to point to a table of multiple maps. Andrii Nakryiko was concerned about how that approach would be represented in the kernel, but the end of the session came before a conclusion was reached. Starovoitov suggested that they should try to find a time to whiteboard possible designs in the future.
