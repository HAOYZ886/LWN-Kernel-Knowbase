---
title: BPF tracing performance
url: https://lwn.net/Articles/978335/
date: "June 18, 2024"
category: BPF
author: "By Daroc Alden June 18, 2024 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Daroc Alden**  
June 18, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

On the final day of the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), the BPF track opened with a series of sessions on improving the performance and flexibility of probes and other performance-monitoring tools, in the kernel and in user space. Jiri Olsa led two sessions about different aspects of probes: making the API for BPF programs attached to a probe more flexible, and making user-space probes more efficient. 

Olsa introduced an improvement to [ kprobes](<https://docs.kernel.org/trace/kprobes.html>); he [ posted](<https://lwn.net/ml/all/20240430112830.1184228-1-jolsa@kernel.org/>) a new way to attach a single program to the entry and exit hooks for a function. This is already technically possible, but it's a pain to work with, because the BPF program has no way to match entries and exits up with one another. Olsa's new API, called kprobe_multi, will give the BPF program a cookie to match calls to the entry and exit hooks with each other, as well as allowing the entry hook to request that the exit hook be skipped if the event is not something the BPF program is interested in. 

The API is already merged into the kernel and supported for kprobes and [ fprobes](<https://docs.kernel.org/trace/fprobe.html>). User-space probes are a harder problem, but Olsa is working on getting a version of his new API for those upstream as well. There are also some improvements he is planning for the kernel side; he would like to reimplement fprobes on top of the [function-graph](<https://lwn.net/ml/all/20181122012708.491151844@goodmis.org/>) (fgraph) mechanism. Since kprobe_multi is built on top of fprobe, this would also change how that API is implemented internally. 

The reason to refactor the design like this is to consolidate the different mechanisms in the kernel that trace returns from function calls, Olsa said. Currently, there are essentially two parallel implementations: fprobe and fgraph. The fgraph code maintains a shadow stack for each kernel task that it uses to keep track of up to 16 trace functions. The shadow stack serves as an alternative to the rethook mechanism that fprobes currently use. It would require one extra page per process, allocated when the process is first traced, but it would mean kprobe_multi programs, fprobes, and fgraph code could all share the same mechanism. 

One audience member asked whether the limit of 16 simultaneous traced functions in fgraph meant that there would only be up to 16 kprobe_multi programs. Olsa clarified that no, the fprobe instances for multiple kprobe_multi programs could share an fgraph slot. One of the changes he has made is to introduce a hash table that is used to make that mapping many-to-one. 

Another person questioned whether implementing kprobe_multi using fprobes, that are themselves implemented with fgraph, which is in turn built on ftrace, was really necessary. Olsa explained that there had been a lot of grumbling about having two separate tracing implementations in the kernel, and that consolidating them actually made things simpler. The audience member then asked whether having so many layers added too much overhead from indirect calls. Another person suggested doing what tracepoints already do to solve excessive indirection — patch the traced code to use a [ static call](<https://lwn.net/Articles/815908/>) when there is only one relevant tracepoint. 

Overall, there were some doubts about whether the refactoring Olsa proposed was actually warranted, but not much doubt that the kprobe_multi interface was a useful addition. 

#### Faster uprobes

Olsa also led the next session, this time focused on making user-space probes faster. To set a uprobe, the user specifies an inode and an offset. When the kernel loads that file into memory (or has it already loaded), it patches the given offset with a breakpoint instruction that triggers the probe. The exact details however, are specific to a given architecture, so Olsa focused only on x86. 

After executing the BPF program attached to the probe, the kernel then needs to execute the original instruction that was replaced. Some instructions can be emulated in the kernel, such as moving values between registers, performing arithmetic, or so on. Other instructions need to be executed in their original context, so the kernel will restore the original instruction, set the CPU's debug flag to single-step instructions, return to the user space program, and then put the breakpoint instruction back and disable the debug flag. 

Some probes are handled a bit differently. Return probes use a return trampoline approach: overwriting the return address on the stack with the address of a trampoline that executes a breakpoint instruction and then returns to the original address. In either case, both approaches add a certain amount of overhead to uprobes; overhead that Olsa would like to minimize. 

One audience member asked whether these techniques were compatible with user-space applications that do non-standard things like perform their own tracing using a shadow stack. Olsa answered that this would cause problems. For example, programs written in Go have a shadow stack that would notice return probes being inserted and crash the application. 

Olsa then explained the benchmarks that he was using to actually measure the overhead of uprobes. He focused on benchmarks involving three different instructions, since those are all common cases and use slightly different implementations: a `nop` instruction, a `push` instruction, and a `ret` instruction. For each one, he patched in a uprobe over the instruction, and then measured the time it took to execute the uprobe. To speed return probes up, Olsa would like to replace the breakpoint instruction with a new `uretprobe()` system call. On x86, system calls are generally faster than triggering a breakpoint — about a 31% speedup on Intel CPUs, and a 10% speedup on AMD CPUs in this case, he noted. The audience member wondered if the difference was because of the speculative-execution side-channel mitigations required by each platform, since Intel needs mitigations for both Spectre and Meltdown, but AMD is only affected by Spectre. 

Other probes will require a different approach. Unlike the return-probe work, which has been implemented and measured, Olsa only has prototypes and ideas for the other probes. One idea is to replace the use of a breakpoint instruction with a jump to a trampoline that makes a system call. If it works, it should provide a roughly equivalent speedup. Unfortunately, there are a lot of complications with the idea. 

The shortest usable jump instruction on x86 is five bytes, compared to one byte for the breakpoint instruction, meaning that patching to include the jump instruction could actually overwrite multiple instructions. Additionally, five-byte updates aren't atomic; if another thread were in the middle of executing the patched code while the kernel was editing it to deal with instructions that require single-stepping, there could be unexpected application crashes. Finally, the five-byte jump instruction only takes a 32-bit address — so using that approach would potentially require multiple trampolines placed throughout a process's address space. 

Even with so many potential problems, there is a strong motivation to solve them. Olsa noted that if you "ignore those issues and just try", it can result in a 250% speedup. So with that context, he wanted to discuss whether anyone had ideas about how to make the approach work. He opened the discussion by suggesting a hybrid approach: write a breakpoint opcode, write the jump offset, and then write the `jmp` opcode. An audience member thought that it may not be safe to do that, pointing out that the only approach to editing code as it is being executed that Intel claims is actually supported is writing a breakpoint opcode. He also noted another problem with using a jump instruction: since it is more than one byte, it's entirely possible that code could jump into the middle of the jump instruction. 

Andrii Nakryiko noted that if they had to deal with synchronization problems anyway, it could make sense to use a longer jump instruction so that multiple trampolines would not be needed. Shung-Hsi Yu asked whether changing the size of the no-op instructions used would require changing the headers used by the user-space tracing infrastructure. Olsa said that it might, but that something needed to change for performance to improve.
