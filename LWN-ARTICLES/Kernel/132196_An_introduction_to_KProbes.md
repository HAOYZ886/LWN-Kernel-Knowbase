---
title: An introduction to KProbes
url: https://lwn.net/Articles/132196/
date: "April 18, 2005"
category: KProbes
author: "April 18, 2005 This article was contributed by Sudhanshu Goswami"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

April 18, 2005

This article was contributed by [Sudhanshu Goswami](<http://idea-factory.blogspot.com>)

### Introduction

KProbes is a debugging mechanism for the Linux kernel which can also be used for monitoring events inside a production system. You can use it to weed out performance bottlenecks, log specific events, trace problems etc. KProbes was developed by IBM as an underlying mechanism for another higher level tracing tool called DProbes. DProbes adds a number of features, including its own scripting language for the writing of probe handlers. However, only KProbes has been merged into the standard kernel. 

In this article I will describe the implementation of KProbes as present in the 2.6.11.7 kernel. KProbes heavily depends on processor architecture specific features and uses slightly different mechanisms depending on the architecture on which it's being executed. The following discussion pertains only to the x86 architecture. This article assumes a certain familiarity with the x86 architecture regarding interrupts and exceptions handling. KProbes is available on the following architectures however: ppc64, x86_64, sparc64 and i386.

A kernel probe is a set of handlers placed on a certain instruction address. There are two types of probes in the kernel as of now, called "KProbes" and "JProbes." A KProbe is defined by a pre-handler and a post-handler. When a KProbe is installed at a particular instruction and that instruction is executed, the pre-handler is executed just before the execution of the probed instruction. Similarly, the post-handler is executed just after the execution of the probed instruction. JProbes are used to get access to a kernel function's arguments at runtime. A JProbe is defined by a JProbe handler with the same prototype as that of the function whose arguments are to be accessed. When the probed function is executed the control is first transferred to the user-defined JProbe handler, followed by the transfer of execution to the original function. The KProbes package has been designed in such a way that tools for debugging, tracing and logging could be built by extending it.

![\[KProbes architecture\]](https://static.lwn.net/images/ns/kernel/KProbesArchitecture.png) The figure to the right describes the architecture of KProbes. On the x86, KProbes makes use of the exception handling mechanisms and modifies the standard breakpoint, debug and a few other exception handlers for its own purpose. Most of the handling of the probes is done in the context of the breakpoint and the debug exception handlers which make up the KProbes architecture dependent layer. The KProbes architecture independent layer is the KProbes manager which is used to register and unregister probes. Users provide probe handlers in kernel modules which register probes through the KProbes manager.

### KProbes Interface

The data structures and functions implementing the KProbes interface have been defined in the file `<linux/kprobes.h>`. The following data structure describes a KProbe.

```
struct kprobe {
        struct hlist_node hlist;                    /* Internal */
        kprobe_opcode_t addr;                       /* Address of probe */
        kprobe_pre_handler_t pre_handler;           /* Address of pre-handler */
        kprobe_post_handler_t post_handler;         /* Address of post-handler */
        kprobe_fault_handler_t fault_handler;       /* Address of fault handler */
        kprobe_break_handler_t break_handler;       /* Internal */
        kprobe_opcode_t opcode;                     /* Internal */        
        kprobe_opcode_t insn[MAX_INSN_SIZE];        /* Internal */
    };
```

Let's first talk about registering a KProbe. Users can insert their own probe inside a running kernel by writing a kernel module which implements the pre-handler and the post-handler for the probe. In case a fault occurs while executing a probe handler function, the user can handle the fault by defining a fault-handler and passing its address in struct kprobe. The prototypes for these are defined as below.

```
typedef int (*kprobe_pre_handler_t)(struct kprobe*, struct pt_regs*);
    typedef void (*kprobe_post_handler_t)(struct kprobe*, struct pt_regs*, 
                  unsigned long flags);
    typedef int (*kprobe_fault_handler_t)(struct kprobe*, struct pt_regs*, 
                 int trapnr);
```

As can be seen the pre-handler and the post-handler both receive a reference to the probe as well as the registers saved for the context in which the probe was hit. These values can be used in the pre-handler or post-handler or if required, they can be modified before returning control to the subsequent instruction. This also means that the same handlers can be used for multiple probe locations. The `flags` parameter is currently unused. The `trapnr` parameter (for the fault handler function) contains the exception number which occurred while handling the KProbe. A user defined fault handler can return 0 to let KProbe handle the fault further. It returns 1 if it has handled the fault and wants to let the execution of the probe handler continue. 

Note that currently the pre-handler cannot be `NULL` for a probe, although the use of post-handler is optional. This is considered a bug since there may be cases where the pre-handler may not be required but a post-handler is needed. In such situations the user will still have to define a pre-handler. Another bug (which can oops the kernel) is related to probes which are activated on the `ret/lret` instructions. Yet another bug is related to probes activated on `int3` instructions. All of these problems should be fixed in the 2.6.12 release of the kernel. However, these bugs can be easily avoided so they do not present any serious issues for someone who wants to use KProbes immediately without applying patches. 

The KProbe registration functions are defined as shown below.

```
int register_kprobe(struct kprobe *p);
    int unregister_kprobe(struct kprobe *p);
```

The registration function takes a reference to the KProbe structure describing the probe. Note that the user's module which registers the probe should keep a reference to the structure until the probe is unregistered. Since access to KProbes is serialized, a probe can be registered or unregistered anytime except from inside the probe handlers themselves, which will deadlock the system. This is because probe handlers execute after the spinlock used for locking KProbes has been acquired. The same spinlock is locked just before unregistering the probe. So if an attempt is made to unregister a probe inside a probe handler the same path will try to lock the spinlock twice. 

Multiple probes cannot be placed on the same address as of now. However, a [patch](<http://marc.theaimsgroup.com/?l=linux-kernel&m=111321506232570&w=2>) has been submitted to the kernel mailing list which allows multiple probes to be registered at the same address through another interface. It might be included in the next release of the kernel. Until then, if such an attempt is made `register_kprobe()` returns `-EEXIST`.

JProbes are used to give access to a function's arguments at runtime. This is achieved by providing a JProbe handler with the same prototype as that of the function being probed. At runtime, when the original function is executed, control is transferred to the JProbe handler after copying the process's context. On return from the JProbe handler, the context - consisting of the process's registers and the stack - is restored, so any modifications to the context of the process in the JProbe handler are lost. The execution continues from the point at which the probe was placed with the original saved state. A JProbe is represented by the structure given below.

```
struct jprobe {
        struct kprobe kp;
        kprobe_opcode_t *entry; 	/* user-defined JProbe handler address */
    };
```

The user places the address of the function which will handle this probe in the `entry` field. The `addr` field in `struct kprobe` should be populated with the address of the function whose arguments are to be accessed. The functions used to register and unregister a JProbe are given below.

```
int register_jprobe(struct jprobe *p);
    void unregister_jprobe(struct jprobe *p);
```

The JProbe handler which is written by the user should call `jprobe_return()` when it wants to return instead of the `return` statement. 

### KProbes Manager

The KProbes Manager is responsible for registering and unregistering KProbes and JProbes. The file `kernel/kprobes.c` implements the KProbes manager. Each probe is described by the `struct kprobe` structure and stored in a hash table hashed by the address at which the probe is placed. Access to this hash table is serialized by the spinlock `kprobe_lock`. This spinlock is locked before a new probe is registered, an existing probe is unregistered or when a probe is hit. This prevents these operations from executing simultaneously on a SMP machine. Whenever a probe is hit, the probe handler is called with interrupts disabled. Interrupts are disabled because handling a probe is a multiple step process which involves breakpoint handling and single-step execution of the probed instruction. There is no easy way to save the state between these operations hence interrupts are kept disabled during probe handling.

The manager is composed of these functions which are followed by a simplified description of what they do. These functions are architecture independent. A side-by-side reading of the code in `kernel/kprobes.c` and these steps will clarify the whole implementation.

`void lock_kprobes(void)`
    Locks KProbes and records the CPU on which it was locked

`void unlock_kprobes(void)`
    Resets the recorded CPU and unlocks KProbes

`struct kprobe *get_kprobe(void *addr)`
    Using the address of the probed instruction, returns the probe from hash table

`int register_kprobe(struct kprobe *p)`
    This function registers a probe at a given address. Registration involves copying the instruction at the probe address in a probe specific buffer. On x86 the maximum instruction size is 16 bytes hence 16 bytes are copied at the given address. Then it replaces the instruction at the probed address with the breakpoint instruction.

`void unregister_kprobe(struct kprobe *p)`
    This function unregisters a probe. It restores the original instruction at the address and removes the probe structure from the hash table.

`int register_jprobe(struct jprobe *jp)`
    This function registers a JProbe at a function address. JProbes use the KProbes mechanism. In the KProbe pre_handler it stores its own handler setjmp_pre_handler and in the break_handler stores the address of longjmp_break_handler. Then it registers struct kprobe jp->kp by calling register_kprobe()

`void unregister_jprobe(struct jprobe *jp)`
    Unregisters the struct kprobe used by this JProbe

### What happens when a KProbe is hit?

![\[Kprobe execution diagram\]](https://static.lwn.net/images/ns/kernel/KProbeExecution.png) The steps involved in handling a probe are architecture dependent; they are handled by the functions defined in the file `arch/i386/kernel/kprobes.c`. After the probes are registered, the addresses at which they are active contain the breakpoint instruction (`int3` on x86). As soon as execution reaches a probed address the `int3` instruction is executed, causing the control to reach the breakpoint handler `do_int3()` in `arch/i386/kernel/traps.c`. `do_int3()` is called through an interrupt gate therefore interrupts are disabled when control reaches there. This handler notifies KProbes that a breakpoint occurred; KProbes checks if the breakpoint was set by the registration function of KProbes. If no probe is present at the address at which the probe was hit it simply returns 0. Otherwise the registered probe function is called.
