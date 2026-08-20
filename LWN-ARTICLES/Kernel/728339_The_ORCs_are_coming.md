---
title: The ORCs are coming
url: https://lwn.net/Articles/728339/
date: "July 20, 2017"
category: Stack unwinder
author: "By Jonathan Corbet July 20, 2017"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 20, 2017

There are a few reasons for wanting the ability to get proper stack traces out of the kernel, including profiling, tracing, and debugging kernel crashes. Historically, the kernel's tracebacks have been unreliable for a number of reasons, most of which have been fixed in recent years. Now it seems likely that the 4.14 kernel will include a new mechanism that should put our traceback problems behind us — for now. 

The state of the kernel's call stack can be surprisingly hard to interpret. Normally, it is made up of normal C function calls, but then assembly-language code, interrupts, processor traps, etc. tend to confuse the picture. A confusing stack can, naturally, cause the "unwinder" code that tries to derive the current call chain from that stack's contents to do strange things; as a result, the kernel has long eschewed any sort of complicated unwinding code. For the most part, developers who deal with kernel tracebacks have learned to cope with occasional bad data. 

The live patching effort, though, depends on accurate call-stack information for its consistency model; in short, it needs to be able to tell which functions appear in the call stack of any thread in the system. Getting there involved implementing the [compile-time stack validation](<https://lwn.net/Articles/658333/>) mechanism to ensure that all kernel code keeps the stack in reasonable condition at all times. The final step is a proper unwinder that uses this now-reliable stack information. 

Last May, an attempt to add such an unwinder based on the [DWARF](<https://en.wikipedia.org/wiki/DWARF>) debugging records emitted by the compiler [ran into trouble](<https://lwn.net/Articles/728347/>) when Linus Torvalds saw it. He noted that, the last time this experiment was tried, the unwinder ran into continual problems from changes to assembly-language code or problems with incorrect DWARF records and, as a result, proved to be unmaintainable. Thus, he said: ""I do not ever again want to see fancy unwinders with complex state machine handling used by the oopsing code."" So DWARF, which requires that sort of complexity, did not appear to be a good option. 

That might have been the end of the story, given that Torvalds was firm in his position, but Josh Poimboeuf [mentioned an idea](<https://lwn.net/Articles/728346/>) he had been pondering for a bit. The `objtool` utility that performs stack validation at compile time builds a model of the state of the stack at every point in the built kernel. Perhaps, he thought, `objtool` could emit the debugging records to make that information available to the unwinder in a format rather simpler than DWARF. The result could be a more reliable unwinder using a more efficient data format that, importantly, is fully under the control of the kernel community and, one would hope, relatively unlikely to break. 

Two months or so later, the result is the [ORC unwinder](<https://lwn.net/Articles/727553/>). The name ostensibly stands for "oops rewind capability", though it's obviously a play on DWARF (which, in turn, is a play on the ELF executable format). The new ORC format is simple at its core; it is based on this structure: 

```
struct orc_entry {
    	s16		sp_offset;
    	s16		bp_offset;
    	unsigned	sp_reg:4;
    	unsigned	bp_reg:4;
    	unsigned	type:2;
        };
```

The purpose of an `orc_entry` structure is to tell the unwinder code how to orient itself on the stack. There is one of these structures associated with each executable address in the kernel, along with a simple data structure allowing the unwinder to find the correct entry given a program-counter address. 

The interpretation of the structure depends on the `type` field. If it is `ORC_TYPE_CALL`, the code is running within a normal C-style call frame, and the beginning of that frame can be found by adding the `sp_offset` value to the value found in the register indicated by `sp_reg`. If, instead, `type` is `ORC_TYPE_REGS`, then that sum points to a `pt_regs` structure describing the processor (and stack) state prior to a system call. Finally, `ORC_TYPE_REGS_IRET` says that `sp_reg` and `sp_offset` can be used to find a return frame for a hardware interrupt. Those three possibilities appear to be enough to describe any situation that will be encountered, at least on the x86 architecture. (The `bp_reg` and `bp_offset` fields don't appear to have much use in the current implementation). 

The resulting mechanism is far simpler than the DWARF mechanism. Among other things, that means it's quite a bit faster — a factor of at least 20x is claimed. Unwinding performance may not matter much when responding to a kernel oops, but it's a big deal for function tracing and profiling. The ORC approach is also claimed to be more reliable than telling the compiler to use frame pointers, and it doesn't suffer from the significant performance hit that frame pointers bring with them. And, as noted above, the ORC format is entirely under the control of the kernel community, so it shouldn't break with new compiler versions and, if it does, kernel developers can fix it. 

Of course, it's hard to predict just how creative the compiler developers of the future may be when it comes to breaking call-stack information. Poimboeuf acknowledges that risk in the patch posting, but notes that: 

If newer versions of GCC come up with some optimizations which break objtool, we may need to revisit the current implementation. Some possible solutions would be asking GCC to make the optimizations more palatable, or having objtool use DWARF as an additional input, or creating a GCC plugin to assist objtool with its analysis. 

The other disadvantage is that the ORC format takes more space than DWARF, occupying 1MB or so of extra memory. Poimboeuf [suggested](<https://lwn.net/Articles/728352/>) that the memory use could be reduced if it turns out to be a real problem. ""However, it will probably require sacrificing some combination of speed and simplicity"". 

Torvalds has not yet made his feelings known regarding the ORC patches, though he had in the past [indicated](<https://lwn.net/Articles/728353/>) that he thought the combination of `objtool` and a simpler format might work. Ingo Molnar, meanwhile, has [applied the patches to the tip tree](<https://lwn.net/Articles/728354/>), indicating that they are likely to show up in a 4.14 pull request. So, barring last-minute problems, the multi-year effort to get a reliable stack unwinder in the kernel may be close to completion.
