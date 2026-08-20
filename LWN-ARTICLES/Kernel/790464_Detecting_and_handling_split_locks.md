---
title: Detecting and handling split locks
url: https://lwn.net/Articles/790464/
date: "June 7, 2019"
category: "Architectures-x86"
author: "June 7, 2019 This article was contributed by Marta Rybczyńska"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

June 7, 2019

This article was contributed by Marta Rybczyńska

The Intel architecture allows misaligned memory access in situations where other architectures (such as ARM or RISC-V) do not. One such situation is atomic operations on memory that is split across two cache lines. This feature is largely unknown, but its impact is even less so. It turns out that the performance and security impact can be significant, breaking realtime applications or allowing a rogue application to slow the system as a whole. Recently, Fenghua Yu has been working on detecting and fixing these issues in the [split-lock patch set](<https://lwn.net/ml/linux-kernel/1556134382-58814-1-git-send-email-fenghua.yu%40intel.com/>), which is currently on its eighth revision.

#### From misaligned memory accesses to split locks

Misaligned memory access occurs when the processor accesses memory at an address that not aligned to the type of the operand, such as an eight-byte operation that accesses a four-byte-aligned variable. Reading four bytes from address `0x1008` is aligned, for example, while the same operation from `0x1006` is not. Misaligned accesses can cause varying behavior on different architectures, including correct and performant operation, exceptions that stop the processor, or incorrect results. 

Misaligned accesses may incur a performance penalty even if the processor transparently handles them. For example, a misaligned access may be split by the CPU into two separate memory operations. Another possibility is the processor generating an exception that is silently handled by the kernel. Portable and high-performance applications should avoid misaligned accesses; the kernel's [code guidelines](<https://www.kernel.org/doc/Documentation/unaligned-memory-access.txt>) state that developers should assume natural alignment requirements on all platforms.

A special type of a misaligned access is one that crosses two cache lines, possibly causing the processor to have to fetch multiple lines before performing the operation. Things get more complicated when an atomic operation is being performed and the processor must ensure that the data involved is seen consistently and correctly while the operation is executed. Intel platforms support atomic accesses that are split across two cache lines; such an operation is called a "split lock".

With a split lock, the value needs to be kept coherent between different CPUs, which means assuring that the two cache lines change together. As this is an uncommon operation, the hardware design needs to take a special path; as a result, split locks may have important consequences as described in the cover letter of Yu's patch set. Intel's choice was to lock the whole memory bus to solve the coherency problem; the processor locks the bus for the duration of the operation, meaning that no other CPUs or devices can access it. The split lock blocks not only the CPU performing the access, but also all others in the system. Configuring the bus-locking protocol itself also adds significant overhead to the system as a whole. 

On the other hand, if the atomic operation operand fits into a single cache line, the processor will use a less expensive cache lock. This all means that developers may increase performance and avoid split locks by actions like simply correctly aligning their variables.

#### Split lock consequences

Yu explained the use cases that motivated this work: hard realtime, cloud computing, and avoiding a security hole. The most important one seems to be related to systems that run hard realtime applications on some cores and normal-priority processes on other cores. Split locks may cause the hard realtime requirements to be broken, as the bus locking caused by split locks executed by the regular code blocks memory accesses by the realtime code. Yu noted that, until now, such complex realtime applications could not be supported for exactly this reason: 

To date the designers have been unable to deploy these solutions as they have no way to prevent the "untrusted" user code from generating split lock and bus lock to block the hard real time code to access memory during bus locking. 

In the cloud case, one user process from a guest system may block other cores from accessing memory and cause performance degradation across the whole system. In a similar way, malicious code may try to slow down the system deliberately in a denial-of-service attack.

#### Solutions

Intel processors, starting with the upcoming Tremont generation, will be able to generate an exception (called "Alignment Check" or `#AC`) when a split lock is detected. Earlier processors support only an event counter for debugging purposes (exposed by an event counter called `sq_misc.split_lock` in perf), but do not allow immediate action from the system. Yu's work is based on this new capability.

The correct response to split locks, including what to do when they are detected while the system firmware is running, was the subject of [some discussion](<https://lwn.net/ml/linux-kernel/6fbff2df-96c9-db51-371b-a23d1a84d070@intel.com/>) during the review of the earlier version of the patch set. The implementation in the current version concentrates on detection of the problem.

If a split-lock event happens in the kernel itself, it issues a warning and disables the detection on the current CPU. After the warning, the faulty instruction will execute and the system will continue — whether the system should go on or panic was one of the main topics of discussion. The rationale is that a split lock in the kernel is a bug and should be fixed, but the bug is not so severe that the kernel should be made to panic. 

The situation is different for user processes, which will be sent a fatal (by default) `SIGBUS` signal. The issue will need to be fixed before that program can run successfully. Something similar happens when a split lock is created by the system's firmware: the system will simply hang at that point. The developers decided on this handling because they were afraid that otherwise [the firmware would never be fixed](<https://lwn.net/ml/linux-kernel/alpine.DEB.2.21.1806221125240.2402@nanos.tec.linutronix.de/>).

Split-lock detection is enabled by default when it is supported by the hardware. However, system administrators have control over the feature: they can use a new kernel parameter (`nosplit_lock_detect`) at boot time to disable it. There is also a sysfs interface to disable it at runtime at `/sys/devices/system/cpu/split_lock_detect`. 

The patch set also includes support for KVM; it emulates the register in guests, exposing the property. The host system will have the feature enabled by default. What to do with the guests was discussed at multiple occasions; the [agreed-on solution](<https://lwn.net/ml/linux-kernel/alpine.DEB.2.21.1904280903520.1757@nanos.tec.linutronix.de/>) that will show up in the next iteration is to enable it in guests when the host kernel has it enabled. It means that if the host kernel has split-lock detection enabled and a guest triggers the exception, it will be stopped. On the other hand, if the host kernel has it disabled, the guest may choose to enable and use it, but is not required to.

#### Further work

The work has been through multiple iterations at this point and has received regular comments from the kernel developers, including Thomas Gleixner and Ingo Molnar. It has still some issues pending, but at the current pace it should show up in the mainline kernel before too long.
