---
title: Expanding seccomp
url: https://lwn.net/Articles/441232/
date: "May 4, 2011"
category: "Security-seccomp"
author: "By Jake Edge May 4, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
May 4, 2011

Sandboxing processes such that they cannot make "dangerous" system calls is an attractive feature that has already been implemented in a limited way for Linux with seccomp. Two years ago, we [looked at a proposal to expand seccomp](<https://lwn.net/Articles/332974/>) to allow more fine-grained control over which system calls would be allowed. That proposal has been mostly dormant since then, but was recently resurrected after incorporating some of the suggestions made at that time. The reaction to the current proposal so far seems positive, and it might just be gaining some traction that the previous patchset lacked. 

Seccomp (from "secure computing") is enabled via a `prctl()` call and, once enabled, restricts the process from making any further system calls beyond `read()`, `write()`, `exit()`, and `sigreturn()`--any other system call will abort the process. That creates a pretty secure sandbox, but it is also extremely limited as there are other things that developers might want to do from within such a sandbox. In fact, the Chromium browser has [gone to great lengths](<https://lwn.net/Articles/347547/>) to implement its own sandbox that uses seccomp, but expands the range of legal system calls through some contortions. 

That led Adam Langley of the Chromium team to [propose](<https://lwn.net/Articles/332438/>) adding a bitmask of allowable system calls for a new seccomp mode. That would have allowed processes to make a binary choice (allowed or disallowed) for each system call. At the time, Ingo Molnar [suggested using the Ftrace filter code](<https://lwn.net/Articles/332986/>) to make the interface even more flexible by allowing filters to be applied to the system call arguments. Essentially, that would make for three choices for each system call: enabled, disabled, or filtered. 

Fast-forward to today, and that is what a [patchset](<https://lwn.net/Articles/440454/>) from Will Drewry implements. It should come as no surprise that Molnar was [pleased](<https://lwn.net/Articles/441365/>) to see his idea result in working code: ""Ok, color me thoroughly impressed - AFAICS you implemented my suggestions [...] and you made it work in practice!"". Eric Paris was likewise [impressed](<https://lwn.net/Articles/441376/>), noting that an expanded seccomp could be used for QEMU. Molnar and Paris did not agree about replacing the LSM approach using filters, but that was something of an aside. Serge E. Hallyn also [pointed out](<https://lwn.net/Articles/441441/>) that the new feature would be useful for containers: ""to try and provide some bit of mitigation for the fact that they are sharing a kernel with the host"". 

The proposed interface, which is likely to change based on comments in the thread, looks like: 

```
const char *filters[] =
          "sys_read: (fd == 1) || (fd == 2)\n"
          "sys_write: (fd == 0)\n"
          "sys_exit: 1\n"
          "sys_exit_group: 1\n"
          "on_next_syscall: 1";
        prctl(PR_SET_SECCOMP, 2, filters);
```

That example is taken from Drewry's [documentation file](<https://lwn.net/Articles/441368/>) that accompanies the patches. It would allow reading from two file descriptors (1 and 2) and writing to one (0), while allowing any calls to the two other system calls listed. The `on_next_syscall` means that the rules would not be enforced until after one more system call is made. That would allow a parent to `fork()`, set up the seccomp sandbox in the child process, then exec a new program which would be governed by the new rules. 

That `on_next_syscall` piece drew a few comments. As it turns out, there are really only two cases that need to be handled, either the rules should go into effect immediately (for a process that wants to restrict itself before handling untrusted input for example), or they should go into effect after an exec (for a parent that is spawning an untrusted child). Making the "after exec" case the default, while still allowing a process to request immediate application, seems to be the way things are headed. 

There were also questions about using kernel-internal symbol names like `sys_read`. Exporting those as a kernel ABI is not likely to pass muster, as it might restrict the option of changing those function names down the road--or require a messy compatibility layer if they did change. Drewry wanted to avoid using the system call numbers as Langley's original patch did, but as Frederic Weisbecker [pointed out](<https://lwn.net/Articles/441397/>), those numbers are already part of the kernel ABI. Drewry is planning to make that switch and users of the interface will need to use the `unistd.h` header file or a library to map system call names to numbers. 

The patches also modify the `/proc/PID/status` file to output any existing filters that are applied to the process. Given that most applications that read that file don't need the extra information, though, Motohiro Kosaki [suggested](<https://lwn.net/Articles/441404/>) that seccomp get its own file. Drewry's plan is to provide that information in the `/proc/PID/seccomp_filter` file instead, and remove it from `status`. 

Since it uses the Ftrace infrastructure and hooks, the new seccomp mode only works for those system calls that have Ftrace events associated with them. Using one of those non-instrumented system calls in the filters will result in an `EINVAL` from the `prctl()` call. Enabling `CONFIG_SECCOMP_FILTER` (which depends on `CONFIG_FTRACE_SYSCALLS`) will allow the use of the new mode. 

Overall, Drewry has been very receptive to suggestions for changes, and the feedback to the concept has been pretty uniformly positive. Molnar suggested breaking out the Ftrace filter engine further--beyond the minimal changes that Drewry's patches make--so that it would be available for more widespread use in the kernel. Molnar does wonder whether Linus Torvalds or Andrew Morton might object to more use of the filter mechanism, however: ""are you guys opposed to such flexible, dynamic filters conceptually? I think we should really think hard about the actual ABI as this could easily spread to more applications than Chrome/Chromium."" So far, neither has spoken up one way or the other. 

Currently it would seem that Drewry is off working on the next revision of the patchset, and it certainly doesn't seem like anything that would be merged in the upcoming 2.6.40 cycle. As Molnar notes, the ABI needs to be carefully thought-out, there are still some RCU issues that are being discussed, and it probably needs some soaking time in the -next tree, but barring some major complaint cropping up, it's a feature that will likely make its way into the mainline relatively soon. While that won't allow Chromium to immediately ditch its complicated sandboxing arrangement, it may well be able to do so a few years down the road. Other applications will benefit from an expanded seccomp as well.
