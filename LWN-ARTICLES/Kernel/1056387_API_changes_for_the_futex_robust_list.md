---
title: API changes for the futex robust list
url: https://lwn.net/Articles/1056387/
date: "February 4, 2026"
category: Futex
author: "By Jake Edge February 4, 2026 LPC"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
February 4, 2026

* * *

[LPC](<https://lwn.net/Archives/ConferenceIndex/#Linux_Plumbers_Conference-2025>)

The [robust futex kernel API](<https://docs.kernel.org/locking/robust-futex-ABI.html>) is a way for a user-space program to ensure that the locks it holds are properly cleaned up when it exits. But the API suffers from a number of different problems, as André Almeida described in a [session](<https://lpc.events/event/19/contributions/2108/>) in the ["Gaming on Linux" microconference](<https://lpc.events/event/19/sessions/225/#20251213>) at the 2025 Linux Plumbers Conference in Tokyo. He had some ideas for a new API that would solve many of those problems, which he wanted to discuss with attendees; there is a difficult-to-trigger race condition that he wanted to talk about too. 

""Some years ago, I made a new API for futex"", Almeida said to start things off, ""so why not do a new API for robust list as well?"" The [new futex API](<https://lwn.net/Articles/823513/>) that he was referring to was merged for 5.16 in 2022 in the form of the `futex_waitv()` system call ([documentation](<https://www.kernel.org/doc/html/latest/userspace-api/futex2.html>)). Some [further pieces of the futex2 API](<https://lwn.net/Articles/940944/>) were released with Linux 6.7 in 2024. 

[ ![\[André Almeida\]](https://static.lwn.net/images/2026/lpc-almeida-sm.png) ](<https://lwn.net/Articles/1057093/>)

The ABI for games on the [SteamOS distribution](<https://store.steampowered.com/steamos>), where much of the work in gaming on Linux is being done, is Windows on the x86 architecture. The games are mostly built for that ABI, but SteamOS also runs on Arm64, which leads to ""a lot of interesting challenges"". It adds the [FEX emulator](<https://fex-emu.com/>) to run x86 binaries on the Arm64 processor in addition to the [Proton compatibility layer](<https://github.com/ValveSoftware/Proton?tab=readme-ov-file#introduction>) that provides the Windows ABI. That has implications for various kernel areas, including futexes, memory management, and filesystems. 

FEX is a just-in-time (JIT) compiler for turning x86 instructions, for both 32 and 64 bits, into Arm64 machine code. As part of that, when it finds a `syscall` instruction, it needs to translate that to the Arm64 system call, but that does not work well for some x86-32 system calls. The FEX project has a [wiki page describing the problematic calls](<https://wiki.fex-emu.com/index.php/Development:32Bit_Syscall_Woes>), one of which is [`set_robust_list()`](<https://www.man7.org/linux/man-pages/man2/set_robust_list.2.html>). 

`set_robust_list()` is used to avoid problems when a futex holder dies before releasing the lock, which will starve any other threads waiting on it. So, when a thread takes a lock, it can add the lock to the robust list, which is a linked list maintained in user space. The thread informs the kernel where the head of the list is using `set_robust_list()`. The exit path for a thread in the kernel uses that information to wake all of the threads waiting for each futex on the list; it also adds the `FUTEX_OWNER_DIED` mark to each futex. One other wrinkle that he mentioned is that a futex can be put into a "pending" field on the list head while an operation (taking or releasing the lock) is being done, but before the linked list has been updated, so that it can be cleaned up if a crash happens in that window. 

#### Why?

A new API is needed for several reasons, he said. The first is that, unlike x86, Arm64 does not have both 32- and 64-bit system calls, so emulating 32-bit applications is difficult—the "compat" system calls are missing. For example, a 32-bit robust list cannot be handled by the 64-bit system call because it cannot parse the list due to the different pointer size. There is a need for the new interface to allow user space to inform the kernel whether it is a 32- or 64-bit robust list so that the kernel can parse the list correctly. 

Another shortcoming of the existing interface is that only one robust list can be set for a thread, but FEX also wants to use robust futexes. If the application uses them, FEX has to choose which one gets that access. A new interface would provide a way to set multiple list heads for a thread. 

There is currently a limit of 2048 entries on a robust list that will be processed by the kernel, which is meant to avoid getting trapped in an infinite loop. But that limit was never documented as part of the API, so user-space programs are unaware of it, which led to a [bug report](<https://sourceware.org/bugzilla/show_bug.cgi?id=19089>) for the GNU C library. With a new API, either the limit should be documented and exposed as part of the API or it should be made limitless using countermeasures against circular lists, he said. 

The final problem is ""much more interesting"" but ""kind of tricky to explain""; it is a race condition that can occur when a futex is being unlocked. The normal sequence for unlocking a robust futex is as follows: 

  1. The address of the futex is put into the pending slot of the robust list 
  2. The futex is removed from the robust list 
  3. The low-level unlock is done, which clears the futex and wakes any threads waiting on it 
  4. The pending slot is cleared 
Between steps three and four, though, another thread can decide to free the futex because it appears to be the only user of the futex. That thread could then allocate memory in the same location as the former futex. Then the original thread, which is about to perform step four, dies, which causes the kernel to write `FUTEX_OWNER_DIED` in the futex, thus corrupting some random memory. It is difficult to reproduce, but it [does happen](<https://sourceware.org/bugzilla/show_bug.cgi?id=14485>). 

Almeida said that he is unsure how to address this. Perhaps serializing the exit path with all of the [`mmap()` and `munmap()`](<https://www.man7.org/linux/man-pages/man2/mmap.2.html>) calls made by the thread is a possibility. Another idea might be to change the API around the pending field somehow to avoid the race. The previous day he had participated in the [extensible scheduler class (sched_ext) microconference](<https://lpc.events/event/19/sessions/229/#20251212>), which got him thinking that perhaps a specialized scheduler could be written to reproduce the problem reliably; that would help in the fixing process and could be turned into a test case as well. 

#### New API

The API he proposed in the session seems to have evolved somewhat since his [v6 patch set posting](<https://lwn.net/ml/all/20251122-tonyk-robust_futex-v6-0-05fea005a0fd@igalia.com/>) in November 2025 (a few weeks before LPC). It consists of two new system calls: 

```
set_robust_list2(struct robust_list_head *head, unsigned int index,
                         unsigned int cmd, unsigned int flags);
                         
        get_robust_list2(int pid, void **head_ptr,
                         unsigned int index, unsigned int flags);
```

The `index` argument is used to distinguish between different lists so that libraries and applications can have their own lists. The `cmd` argument to `set_robust_list2()` can be `CREATE_LIST_32` (or 64) to create a list of the appropriate "bitness" using the `head` pointer; in that case, the call returns an unused index that is associated with the list. A list can be overwritten using the `SET_LIST_32` (or 64) command by passing the index of interest. The `LIST_LIMIT` command returns the number of lists supported for each task. (All of the command names will presumably have `FUTEX_ROBUST_LIST_CMD_` as part of their full name.) `get_robust_list2()` will simply return the head of the robust list (in `head_ptr`) for a given `pid` and `index`. 

#### Discussion

After that, Almeida opened the floor to questions and comments. Liam Howlett noted that the exit path for robust lists requires a delay to the out-of-memory (OOM) handling in the kernel, so the race condition could be more easily reproduced by setting the OOM-handler delay to zero and triggering an OOM-kill of the task. While that may be true, glibc maintainer Carlos O'Donell said, it does not really lead to a solution to the race, which both he and Rich Felker of the [musl libc project](<https://musl.libc.org/>) have looked at. If there is going to be a new API, it is a ""perfect opportunity"" to sit down and figure out a proper solution and also determine how existing C libraries can transition to the new interface over time. 

""It gets worse"", Howlett said. Tasks that are exiting can be frozen by the control-group subsystem, which means that the OOM handler has to wait potentially forever before it can clean things up. That is another piece that should be unwound as part of the process of creating the new API, he said. 

O'Donell said that it made sense that users of the new API will need to be able to register the number of bits in the structure that is being shared with the kernel. He asked if sizes other than 32 or 64 bits should be considered, but Howlett pointed out that there is an unused `flags` argument in the proposed API, which could be used if needed. 

The conversation turned back to the delay for the OOM handler, which no one seemed to fully understand. O'Donell wondered if it was an attempt to fix the race condition that Almeida is concerned about when it arose in some other context. Howlett said that he believed it was meant to hold off the OOM killer from freeing memory holding the locks before the exit-handling code could process the robust list. Sebastian Siewior said that he was not clear on why the delay was added, either, but would put it on his list to look into. 

There was some further discussion of why and how the OOM-killer delay came about, but the session ran out of time. Interested readers may want to consult the [YouTube video](<https://www.youtube.com/watch?v=BV2DJoXLUrw>) and [slides](<https://lpc.events/event/19/contributions/2108/attachments/1947/4170/robust-futex.pdf>) from the talk. Overall, participants seemed to agree that the new API was needed, and no real complaints about its proposed form were heard, but there are obviously still some details to be worked out before it can go upstream. 

[ I would like to thank our travel sponsor, the Linux Foundation, for assistance with my travel to Tokyo for Linux Plumbers Conference. ]
