---
title: "Kernel support for hardware-based control-flow integrity"
url: https://lwn.net/Articles/900099/
date: "July 11, 2022"
category: "Security-Control-flow integrity"
author: "July 11, 2022 This article was contributed by  Mike Rapoport"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

July 11, 2022

This article was contributed by Mike Rapoport

Once upon a time, a simple stack overflow was enough to mount a code-injection attack on a running system. In modern systems, though, stacks are not executable and, as a result, simple overflow-based attacks are no longer possible. In response, attackers have shifted to control-flow attacks that make use of the code already present in the target system. Hardware vendors have added a number of features intended to thwart control-flow attacks; some of these features have better support than others in the Linux kernel. 

[Control-flow integrity](<https://en.wikipedia.org/wiki/Control-flow_integrity>) (CFI) is a set of techniques intended to prevent control-flow attacks, or at least to reduce the ability of an attacker to hijack the program's control flow. The general idea behind CFI is to label sources and destinations of indirect jumps (such as calls, branches, and return instructions) and verify at run time that the actual jump target matches the label. CFI can be implemented entirely in software, but there are also several hardware mechanisms from different vendors that assist in CFI implementation. 

#### Coarse-grain forward-edge protection

One common way to corrupt a program's control flow is via indirect jumps, such as function calls through a pointer variable. If that pointer can be somehow overwritten, control can be rerouted to a location of an attacker's choosing, with unfortunate results. Just about any program of reasonable size will contain code segments that will do unwanted things if control lands in the middle of them. The term "forward edge" is used to describe attacks on outgoing control flow. Purely software-based protection against forward-edge attacks exists ([forward-edge CFI in LLVM](<https://lwn.net/Articles/898040/>), for example), but hardware-based mechanisms exist as well. 

The arm64 and x86 architectures enable coarse-grain protection of the forward edges using a dedicated `BTI` (arm64) and `ENDBR{32,64}` (x86) instructions, which create a "landing pad" for indirect branches. When an indirect branch is executed, the CPU will check for the landing pad at the target address; if a landing pad is not found, the CPU will trap. Linux supports these protections for the kernel itself on both arm64 and x86, however for now only arm64 supports enabling this feature for user space; the x86 version is still under development. 

While protection of forward edges prevents jumps into the middle of code blocks, it is still possible to find useful code immediately following those landing pads and, of course, there are also the backward edges to exploit. 

#### Return address integrity

[Return-oriented programming](<https://en.wikipedia.org/wiki/Return-oriented_programming>) (ROP) attacks use stack overflows to replace the return address of a function with the address of a machine-code sequence that performs some action the attacker needs and which ends with return instruction. Such a sequence is called "gadget". It is possible to chain multiple gadgets and, as the size of the program under attack grows, it becomes increasingly easy for an attacker to create a chain that performs any desired operation. The term "backward edge" is often used to describe ROP attacks. 

Many architectures store return addresses in a special [link register](<https://en.wikipedia.org/wiki/Link_register>). The non-leaf functions, though, must save the contents of the link register on the stack before calling another function, then restore the link register after that function returns. To protect the on-stack copy of the return address, the arm64 and powerpc architectures allow storing a cryptographically calculated hash alongside the address. These architectures provide special instructions that can generate a random key for crypto operations, create of a hash based on that key and the address value, and validate that the hash matches the value loaded from the stack. 

There are differences in where that hash is stored; on powerpc systems it's saved on the stack next to the pointer it protects, while on arm64 systems the hash occupies unused high bits in the pointer itself. The powerpc and arm64 mechanisms are both supported by GCC and LLVM, but only arm64 has proper support for this feature (called [pointer authentication](<https://lwn.net/Articles/718888/>)) in the Linux kernel. 

Another way to ensure backward-edge integrity is to use a shadow stack to supplement the regular stack. The shadow stack stores the return address for each function that is called; whenever a function returns, the return address on the normal stack is compared to the address on the shadow stack. Execution is only allowed to continue if the two addresses match. Both Intel and AMD implement shadow-stack support in hardware, so that every `CALL` instruction saves the return address to both the normal and the shadow stack, and every `RET` instruction verifies that return addresses match. When the return addresses differ, the CPU generates a control-protection fault. The shadow-stack implementation also affects behavior of other instructions that change the control flow, including `INT`, `IRET`, `SYSCALL`, and `SYSRET`. 

A shadow stack is more demanding to support than pointer authentication. The last version of the patches that enable support for shadow stacks for user space contained 35 patches and was [covered in depth](<https://lwn.net/Articles/885220>) here. Along with enablement of the hardware's shadow-stack functionality and plumbing it into the core kernel, these patches define new kernel APIs that reserve memory for the shadow stack, enable and disable the shadow-stack functionality, and can also lock the state of the shadow-stack features. These APIs are intended to be used by the C library; the overall presence of a shadow stack should be transparent for most applications. These patches have not yet been merged into the mainline kernel, though. 

#### Special cases

Some applications, like [GDB](<https://www.sourceware.org/gdb/>) and [CRIU](<https://criu.org/Main_Page>), need the ability to control the execution of other processes, meaning that they require ways to deal with the shadow stack in a nonstandard way. The GDB debugger, for example, often needs to jump between the stack frames of the program being debugged; it needs to keep the shadow stack in sync with the normal stack as it moves up and down the call chain. 

The contents of the shadow stack can be updated using the `PTRACE_POKEUSER` command to [ptrace()](<https://man7.org/linux/man-pages/man2/ptrace.2.html>), but another `ptrace()` call is then required to update the shadow-stack pointer. There was a proposal to extend `PTRACE_GETREGSET` and `PTRACE_SETREGSET` to support access to the registers controlling the shadow-stack machinery; it was posted as a part of version 11 of the "[Control-flow Enforcement: Indirect Branch Tracking, PTRACE](<https://lwn.net/ml/linux-kernel/20200825002645.3658-1-yu-cheng.yu@intel.com/>)" series, but has not been reposted since then. This interface is used by [Intel's fork of GDB](<https://github.com/intel/gdb>), but what the final form of the kernel API for manipulating shadow-stack control registers will be remains unclear. 

Like GDB, CRIU has an intimate relationship with the processes it checkpoints and restores. The `ptrace()` interfaces intended for GDB are certainly useful for CRIU, but they are not enough. Beside the ability to adjust shadow-stack contents and the shadow-stack pointer, CRIU must be able to restore the shadow stack at the same virtual address as is was at the time of checkpoint. A possible way to achieve that is to extend the proposed `map_shadow_stack()` system call to accept an additional address parameter; when this parameter is not zero, `map_shadow_stack()` will behave similarly to `mmap(MAP_FIXED)` and will attempt to reserve the memory for the shadow stack at the requested location. 

Another issue CRIU has to cope with is the need to restore the shadow stack's contents. Of course, this could be done using `ptrace()`, but that would be slow and would require major changes to the restore logic. A better way is to use a special `WRSS` instruction that lets an application write to its own shadow stack. But this instruction is only available when the shadow stack is enabled, which presents a problem of its own. 

When the GNU C Library loads a program, it enables or disables shadow-stack features for that program based on its ELF header, then locks the feature state permanently. CRIU uses [`clone()`](<https://man7.org/linux/man-pages/man2/clone.2.html>) to create the restored tasks, with the result that they inherit the shadow-stack state from the CRIU control process. So, if CRIU was built with the shadow stack enabled, it wouldn't be able to restore tasks without shadow stack and vice versa. A solution to this was an additional `ptrace()` call to override the shadow-stack feature lock. 

CRIU also needs patches to support pointer authentication because, despite the feature being available for some time now, nobody got around to implementing it in CRIU. 

#### What comes next

We have not yet seen what the final form of the kernel APIs exposed by the shadow-stack implementation will be after it actually lands in the upstream kernel. After that, GDB, CRIU, and maybe other applications that do tricky things with their control flow should be updated to cope with restrictions that shadow stacks pose. And, after the shadow-stack support for user space settles, there is another interesting question to answer: what would it take to support a shadow stack for the Linux kernel and are the complexity and the effort required worth it?
