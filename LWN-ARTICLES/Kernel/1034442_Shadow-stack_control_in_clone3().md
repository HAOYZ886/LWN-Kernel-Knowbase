---
title: "Shadow-stack control in clone3()"
url: https://lwn.net/Articles/1034442/
date: "August 26, 2025"
category: "Security-Control-flow integrity; System calls-clone"
author: "By Jonathan Corbet August 26, 2025"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 26, 2025

Shadow stacks are a control-flow-integrity feature designed to defend against exploits that manipulate a thread's call stack. The kernel first [gained support for hardware-implemented shadow stacks](<https://lwn.net/Articles/926649/>), for the x86 architecture, in the 6.6 release; 64-bit Arm support followed in 6.13. This feature does not give user space much control over the allocation of shadow stacks for new threads, though; a [patch series](<https://lwn.net/ml/all/20250819-clone3-shadow-stack-v19-0-bc957075479b@kernel.org>) from Mark Brown may, after many attempts, finally be about to change that situation. 

As its name suggests, a shadow stack is a sort of copy of a thread's ordinary call stack, but its contents are limited to return addresses. On a system with shadow-stack support, each function call will push the return address onto both the normal and shadow stacks. On return from a function, the return addresses are popped from both stacks and compared; if they do not match, some sort of corruption has occurred and the thread in question is killed. 

The shadow stack is marked specially in the system's page tables and is not writable from user space. A shadow stack must also contain a special, hardware-generated token at the top; this token identifies it as a real shadow stack, and prevents multiple threads from using the same shadow stack. The setup requirements mean that the creation of a shadow stack for a thread must be done by the kernel. 

When the kernel goes to create a thread's shadow stack, there is an immediate question to be answered: how large should that stack be? The creation of the initial shadow stack for a process can be influenced by the process itself, but that is not true for any threads that the process may create thereafter. Whenever a new thread comes into existence, it is given a shadow stack that is the same size as its regular stack as the kernel's best guess for the right size. 

That guess, of course, could be far from the mark. While quite a bit of information — local variables and saved registers, for example — is pushed onto the call stack, the shadow stack only holds return addresses, so there is a good chance that an equally sized shadow stack will be far too large for the thread's needs. If the process creates many threads, the amount of memory wasted by oversized shadow stacks could become significant. There are also (less common) situations, described in the 2023 article, when an equally sized shadow stack could turn out to be too small. 

Either way, there would be value in giving a process a voice in the sizing of shadow stacks for the threads it creates. For some time now, Mark Brown has been working on the ability to control shadow-stack allocation when a thread is created with [`clone3()`](<https://lwn.net/Articles/792628/>); that work was [covered here](<https://lwn.net/Articles/953794/>) in 2023. At that point, the patch set was in its fourth revision; it is now in its 19th iteration, a number of changes have been made, and Brown is expressing hopes that the series will soon be ready to merge. 

Early versions of the series allowed user space to specify the address and size of the desired shadow stack, but that ran into opposition from other developers. Subsequent attempts, which only allowed the size of the shadow stack to be specified, proved to be too limiting. In the end, it was decided to just have the parent process create a shadow stack as it sees fit. So, since version 5, the process calling `clone3()` must first ask the kernel to create the shadow stack for the new thread to use with a call to `map_shadow_stack()`: 

```
sstack = map_shadow_stack(unsigned long addr, unsigned long size,
        			      unsigned int flags);
```

The `addr` and `size` arguments describe where the newly created stack should be placed and how big it should be; setting `addr` to zero leaves the placement decision to the kernel. The `flags` argument must be `SHADOW_STACK_SET_TOKEN` to cause the kernel place the special token at the top of the stack; otherwise the resulting shadow-stack mapping cannot be used for the new thread. This system call will return the virtual address where the stack is actually mapped. 

When the time comes to call `clone3()`, the new `shadow_stack_token` field in [`struct clone_args`](<https://elixir.bootlin.com/linux/v6.16.1/source/include/uapi/linux/sched.h#L47>) must be set to point to the shadow-stack token. This requirement might be a bit unintuitive; the pointer is to the token, _not_ to the base of the stack itself. So code using this feature will look something like this: 

```
struct clone_args args;
        void *ss_addr;
    
        ss_addr = map_shadow_stack(0, 4096, SHADOW_STACK_SET_TOKEN);
        args->shadow_stack_token = ss_addr + 4096 - sizeof(void *);
        pid = clone3(&args, sizeof(args));
```

This code allocates a single, 4096-byte shadow stack, letting the kernel choose where the stack should be placed; it then points `args->shadow_stack_token` at the token that the kernel will have stored at the top of the stack. The thread created with the subsequent `clone3()` call will, if all goes well, be using the newly created shadow stack. Error handling, obviously, has been omitted. 

Brown said, in the cover letter: ""I think at this point everyone is OK with the ABI, and the x86 implementation has been tested so hopefully we are near to being able to get this merged?"" So far, nobody has spoken up to disagree with that idea. In the absence of surprises, this long-under-development addition to the `clone3()` API may finally be headed for the mainline.
