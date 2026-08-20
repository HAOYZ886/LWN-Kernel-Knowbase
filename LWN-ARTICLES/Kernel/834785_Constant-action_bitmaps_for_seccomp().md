---
title: "Constant-action bitmaps for seccomp()"
url: https://lwn.net/Articles/834785/
date: "October 22, 2020"
category: "Security-seccomp"
author: "By Jonathan Corbet October 22, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 22, 2020

The [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>) system call allows user space to load one or more (classic) BPF programs to be run whenever the calling process invokes a system call. Those programs can examine ([to an extent](<https://lwn.net/Articles/799557/>)) the arguments to each call and inform the kernel whether the call should be allowed to proceed or not. This feature is used in a number of containerization solutions (and beyond) as a way of reducing the kernel's attack surface. In some situations, though, using `seccomp()` can result in a significant performance reduction. There are currently two patch sets in circulation that are aimed at reducing the overhead of `seccomp()` for one common use case. 

The argument-inspection feature of `seccomp()` is useful in a number of settings; it can, for example, block a `write()` call to any file descriptor other than the standard output. But many real-world use cases do not take advantage of this capability; instead, they make decisions based only on which system call is being invoked while paying no attention to the arguments to those calls. It turns out that the BPF mechanism is far from optimal for this case, which must be implemented as a long series of comparisons against the system-call number. The overhead of these comparisons can be reduced by using smarter algorithms (checking for the most commonly used system calls first, for example), but there are limits to how fast it can be. This overhead makes every system call slower. 

Much of this work is wasted. If a `seccomp()` configuration of this type allows `read()` once, it will allow it every time, but the kernel must work it out the hard way each time regardless. If there were some way of knowing that a given `seccomp()` filter program allows or denies specific system calls without looking at their arguments, it would be possible to implement those decisions much more quickly. 

#### Optimizing `seccomp()`

In June, Kees Cook [posted a patch](<https://lwn.net/ml/linux-kernel/20200616074934.1600036-1-keescook@chromium.org/>) implementing this sort of optimization. It creates three bitmaps (called `allow`, `kill_thread`, and `kill_process`) within a process; they are indexed by the system-call number. When a system call is intercepted by `seccomp()`, the associated number is used to consult those bitmaps; if the relevant bit is set in a bitmap, the associated action is taken without ever actually running the BPF program. Thus, the bits for always-allowed system calls can be set in the `allow` bitmap; they will then execute far more quickly. 

The trick is setting those bits in the first place. Cook's patch set works by actually executing the loaded BPF program(s) at load time for every supported system call and watching what happens. If, for a given system call, the BPF code does not access the system-call arguments, the kernel can conclude that the result will always be the same for that call and set a bit in the appropriate bitmap. If, instead, the arguments are accessed, the bit for that system call is cleared in all bitmaps; the BPF program will thus be executed on every invocation of that call. 

There is another challenge here: observing whether the BPF program does, in fact, access the system-call arguments. The first version of the patch set did that by placing the arguments in a separate page, running the BPF code, then looking at the page-table entry to see whether the page had been referenced or not. This mechanism worked, but relied on some complex memory-management trickery. 

Jann Horn [had a better idea](<https://lwn.net/ml/linux-kernel/CAG48ez1p=dR_2ikKq=xVxkoGg0fYpTBpkhJSv1w-6BG=76PAvw@mail.gmail.com/>): simply emulate the execution of the BPF program and watch what it does directly. The key observation was that the emulator need not be complete, since programs that only compare system-call numbers tend to be quite simple. Only a small subset of the available instructions would need to be emulated; anything that the emulator does not recognize can be taken as an indication that more complex logic is involved and the bitmap cannot be used. 

On September 21, YiFei Zhu showed up with [a patch series](<https://lwn.net/ml/bpf/cover.1600661418.git.yifeifz2@illinois.edu/>) implementing a constant-action `seccomp()` bitmap using an emulator to determine whether the system-call arguments were being accessed or not. There were a number of other differences from Cook's approach; for example, only the "allow" bitmap is implemented on the understanding that the "deny" cases do not really need to be optimized. Two days later, Cook [posted a new version](<https://lwn.net/ml/linux-kernel/20200923232923.3142503-1-keescook@chromium.org/>) with a rather simpler emulator that is closer to the design first suggested by Horn. Less than one day after that, Zhu returned with [a revised series](<https://lwn.net/ml/linux-kernel/cover.1600946701.git.yifeifz2@illinois.edu/>) with a simplified emulator that borrowed some ideas from Cook's version. 

Cook [described](<https://lwn.net/ml/linux-kernel/202009241253.7D238A4@keescook/>) Zhu's initial patch set as ""significantly over-engineered"" and said that he had rushed out his updated version to show ""how small I would like the emulator to be"" and how the architecture support could be improved. Since then, it would appear that many of the ideas from Cook's implementation have found their way into Zhu's. [Version 5](<https://lwn.net/ml/linux-kernel/cover.1602431034.git.yifeifz2@illinois.edu/>) of Zhu's patch set, posted on October 11, adds 292 lines of code to `kernel/seccomp.c` — compared to 400 in the initial version — while supporting more functionality. Cook has not reposted his work since, suggesting that Zhu's version may be the one that is ultimately merged. 

#### Paths not taken

There is an interesting question to be considered here. Emulating BPF execution and watching what happens does not seem like the most elegant solution to the problem; there are at least two other approaches that could be considered: 

  * The developers writing the `seccomp()` programs surely know what the desired behavior is. A new `seccomp()` API could be created to allow user space to pass the bitmap in directly rather than having to reverse-engineer it in the kernel. 
  * `seccomp()` is one of the few places in the kernel still using the classic BPF dialect. Switching to extended BPF would allow the writing of programs that could make these decisions much more quickly, again without the need to add code to the kernel to guess what the programs do. 

The question of why these approaches have not been explored has seen relatively little discussion as these patch sets were considered. Horn did [note](<https://lwn.net/ml/linux-kernel/CAG48ez3Z5Cxxc47x8gZiy9xgEVPL5gH-g0HNPNqndkAnJCbwKw@mail.gmail.com/>) that creating a new API would require changes in user space to take advantage of it, while the current patch sets will simply make existing programs run more quickly with no changes required. The proposed patches also make no changes to the user-space API for `seccomp()`, meaning that the kernel community is not committing to anything new by adopting them. A new API for loading the bitmask, instead, would have to be supported forever. 

With regard to eBPF, adding that support to `seccomp()` has come up a few times in the past; the last such would appear to be [this patch series](<https://lore.kernel.org/netdev/20180226072651.GA27045@ircssh-2.c.rugged-nimbus-611.internal/>) posted by Sargun Dhillon in 2018. There are a number of obstacles to be overcome before this support will ever make it into the mainline, though. The BPF maintainers [are concerned](<https://lore.kernel.org/netdev/20180226230418.46nczgkh5csakyu7@ast-mbp/>) that use of eBPF in `seccomp()` could constrain the future development of eBPF itself. Security-oriented developers, instead, worry about the extra capabilities and attack surface provided by eBPF; it would not be hard to introduce new vulnerabilities by putting `seccomp()` and eBPF together. There is also the little problem that `seccomp()` filters can be loaded by unprivileged processes, and giving unprivileged code the ability to load eBPF programs is an idea that [has fallen on hard times](<https://lwn.net/Articles/796328/>). 

The end result is that the current patch sets would appear to be the best that is on offer for improved `seccomp()` performance anytime soon. The performance increase that comes with using the bitmap is real, according to some benchmarks included with Zhu's patch set. So one should expect to see this optimization merged, presumably for the 5.11 development cycle.
