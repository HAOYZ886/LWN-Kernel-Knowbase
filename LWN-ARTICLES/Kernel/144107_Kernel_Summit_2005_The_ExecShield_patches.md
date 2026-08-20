---
title: "Kernel Summit 2005: The ExecShield patches"
url: https://lwn.net/Articles/144107/
date: "July 19, 2005"
category: "Security-Security technologies"
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
Arjan van de Ven took half an hour to discuss the set of security-related patches known as "ExecShield." He made an analogy with building security; just as a number of techniques are employed to secure a building, multiple approaches must be taken to computer security. The login process is like a guard at the door; SELinux is like a set of fire doors inside the building, and an intrusion detection system is like an alarm system. In this analogy, ExecShield is the locks on the windows. 

Various components of ExecShield are: 

  * Non-executable data areas. With advances in the hardware area, this issue has just about gone away. The kernel already has support for this sort of protection. 

  * Various runtime checks built into the kernel API. Arjan mentioned checking for double-free errors in particular. Freeing a chunk of memory twice, as it turns out, is a common bug - and it is exploitable. 

  * Address-space randomization techniques. Some of the ExecShield randomization patches were merged into 2.6.12 (see [this Kernel Page article](<http://lwn.net/Articles/121845/>)), but there are some others. The full ExecShield patch can randomize the locations of the program body, `brk()` area, data area, and more. 

  * The gcc FORTIFY_SOURCE option. This feature is currently only available for user-space programs, but a patch adding kernel support is circulating. This option, in particular, can check the correctness of many memory copy operations when the size of the affected buffer is known. It is less useful in the kernel - which keeps few buffers on the stack - but still worth having. 

  * The `-fstack-protector` gcc option. It adds a canary to the stack, and can thus detect stack buffer overruns after they happen (but, with luck, before they do any harm). 

  * Information hiding. The kernel exports information (through `/proc/_pid_ /maps`, for example) which can be highly useful to attackers. In particular, disclosing the locations of memory areas defeats much of the advantage of randomization. So restricting access to that information improves the security of the system. 

  * `/dev/mem` is used primarily by root kits. About the only legitimate user is the X server, which uses `/dev/mem` for access to the frame buffer. Read access to `/dev/mem` is an information leak, and there is no reason for allowing write access to kernel memory at all. 

Arjan estimates that the ExecShield patches mitigate the effects of some 25-30% of reported vulnerabilities in Linux systems. Even so, there was no discussion of merging the remaining ExecShield patches in the near future.
