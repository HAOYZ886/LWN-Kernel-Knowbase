---
title: "Who audits the audit code?"
url: https://lwn.net/Articles/600568/
date: "May 29, 2014"
category: Auditing
author: "By Jonathan Corbet May 29, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 29, 2014

The Linux audit subsystem is not one of the best-loved parts of the kernel. It allows the creation of a log stream documenting specific system events — system calls, modifications to specific files, actions by processes with certain user IDs, etc. For some, it is an ideal way to get a handle on what is being done on the system and, in particular, to satisfy various requirements for security certifications (Common Criteria, for example). For others, it is an ugly and invasive addition to the kernel that adds maintenance and runtime overhead without adding useful functionality. More recently, though, it seems that audit adds some security holes of its own. But the real problem, perhaps, is that almost nobody actually looks at this code, so bugs can lurk for a long time. 

The system call auditing mechanism creates audit log entries in response to system calls; the system administrator can load rules specifying which system calls are to be logged. These rules can include various tests on system call parameters, but there is also a simple bitmask, indexed by system call number, specifying which calls might be of interest. One of the first things done by the audit code is to check the appropriate bit for the current system call to see if it is set; if it is not, there is no auditing work to be done. 

Philipp Kern recently [noticed a little problem](<https://lwn.net/Articles/600573/>) with how that code works with the x32 ABI. When code running under that ABI invokes a system call, it does not use the normal system call numbers defined by the x86 architecture; instead, x32 system calls (which require compatibility handling for some parameters) are marked by setting an additional bit (0x40000000) in that number. The audit code fails to remove that bit before checking the system call number in its bitmask; as one might imagine, the results are not as one might wish. Philipp included a patch to strip out the x32 bit, but it turns out that the problem is a bit bigger than that. 

Andy Lutomirski, in looking at Philipp's patch, realized that the code wasn't just failing to strip out one bit; there are, in fact, no bounds checks on the system call number at all. User space can pass in any system call number it wants, and the kernel will use that number to index into its bitmask array; the result for a sufficiently large system call number is a predictable kernel oops. Andy also [suggested](<https://lwn.net/Articles/600575/>) that this failure could be used to determine the value of specific bits in kernel space, leading to an information-disclosure vulnerability. 

Andy submitted [a patch](<https://lwn.net/Articles/600577/>) to fix this particular problem, but he didn't stop there. He has come to the conclusion that the audit subsystem is beyond repair, so his patch marks the whole thing as being broken, making it generally inaccessible. He cited a number of problems beyond this security issue: it hurts performance even when it is not being used, it is not (in his mind) reliable, it has problems with various architectures, and ""its approach to freeing memory is terrifying"". All told, Andy [said](<https://lwn.net/Articles/600578/>), we're better off without it: 

In summary, the code is a giant mess. The way it works is nearly incomprehensible. It contains at least one severe bug. I'd love to see it fixed, but for now, distributions seem to think that enabling CONFIG_AUDITSYSCALL is a reasonable thing to do, and I'd argue that it's actually a terrible choice for anyone who doesn't actually need syscall audit rules. And I don't know who needs these things. 

It is unsurprising that Eric Paris, who maintains the audit code, [disagrees](<https://lwn.net/Articles/600594/>) with this assessment. His point of view is that this is just another bug in need of fixing; it does not indicate any systemic problem with the audit code. 

It is telling, though, that this particular vulnerability has existed in the audit subsystem almost since its inception. The audit code receives little in the way of review; most kernel developers simply turn it off for their own kernels and look the other way. But this subsystem is just the sort of thing that distributors are almost required to enable in their kernels; some users will want it, so they have to turn it on for everybody. As a result, almost all systems out there have audit enabled (look for a running `kauditd` thread), even though few of them are using it. These systems take a performance penalty just for having audit enabled, and they are vulnerable to any issues that may be found in the audit code. 

If audit were to be implemented today, the developer involved would have to give some serious thought, at least, to using the tracing mechanism. It already has hooks applied in all of the right places, but those hooks have (almost) zero overhead when they are not enabled. Tracing has its own filtering mechanism built in; the addition of [BPF-based filters](<https://lwn.net/Articles/599755/>) will make that feature more capable and faster as well. In a sense, the audit subsystem contains yet another kernel-based virtual machine that makes decisions about which events to log; using the tracing infrastructure would allow the removal of that code and a consolidation to a single virtual machine that is more widely maintained and reviewed. 

The audit system we have, though, predates the tracing subsystem, so it could not have been based on tracing. Replacing it without breaking users would not be a trivial job, even in the absence of snags that have been glossed over in the above paragraph (and such snags certainly exist). So we are likely stuck with the current audit subsystem (which will certainly not be marked "broken" in the mainline kernel) for the foreseeable future. Hopefully it will receive some auditing of its own just in case there are more old surprises lurking therein.
