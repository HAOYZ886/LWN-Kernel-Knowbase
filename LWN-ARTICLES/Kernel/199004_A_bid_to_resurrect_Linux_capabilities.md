---
title: A bid to resurrect Linux capabilities
url: https://lwn.net/Articles/199004/
date: "September 10, 2006"
category: "Capabilities; Security-Security technologies"
author: 
---

Back in 1998, as the 2.1 kernel went into yet another feature freeze, the capabilities feature was merged. Capabilities split the power of the root account into a set of privileges, each of which can be granted or withheld independently of the others. A process which needs to be able to bind to a privileged port number, for example, could be given that ability without simultaneously enabling it to override file permissions, kill other processes, or exceed resource limits. Proponents of capabilities have long seen a world where the root account no longer exists and all tasks have the minimum level of privilege they need to get their jobs done. A system organized in this way, it is thought, would be more secure. 

The world is full of Linux distributions, many of which are oriented toward higher levels of security. But, to your editor's knowledge, nobody has ever put together a successful, capability-based distribution. There are many reasons for this lack of implementations, including the fact that nobody has really figured out a way to administer a system with a couple dozen more security-related bits attached to every executable file. But one should also not overlook the fact that, from the 2.1.x days to now, there has never been a Linux kernel where capabilities actually worked as intended. 

Part of the problem is an incomplete implementation: no patch which attaches capability masks to files has ever been merged. But the kernel has also never implemented capability inheritance - what happens to the capability bits when a process executes a new program - in a correct manner. For some time now, in fact, capability inheritance has been disabled completely. Without inheritance, the full capability model cannot work. So the use of capabilities in Linux systems has been limited to a very small number of programs which have been coded to drop the capabilities they do not need. 

David Madore has set out to change that state of affairs with [a set of patches](<http://lwn.net/Articles/198977/>) to fix up capability support. This patch set does a few things, the first of which being to expand the capability set from 32 to 64 bits. Current kernels have 31 capabilities defined, so it is not especially hard to imagine needing more in the future. That need could become pressing if anybody ever gets serious about splitting the catch-all `CAP_SYS_ADMIN` capability into several smaller privileges. 

This patch uses some of those new bits from the outset for a set of "regular capabilities" which all processes are normally expected to have. These capabilities include the ability to use `fork()` or `exec()`, the ability to open files and to write to files, the ability to use `ptrace()`, and the ability to increase privilege by running a setuid program. The idea here is that processes running in security-relevant settings can drop those capabilities if they are not needed, making it harder to exploit any vulnerabilities in those processes. 

The core of the patch, however, is the implementation of capability inheritance. Understanding this part requires just a bit of background. As it happens, while one can talk about the capabilities possessed by a process, each process in Linux has three separate capability masks. The **permitted** set is all of the capabilities that the process is allowed to have. But capabilities cannot be used unless they are set in the **effective** set, is a subset of the permitted set. Finally, each process has an **inheritable** set, listing the capabilities (again, a subset of the permitted set) which can be passed on to any program run with `exec()`. Processes can adjust the effective and inheritable sets at any time (within the bounds of the permitted set), but the permitted set cannot be expanded. 

In a capability-based system, executable files also have a set of three capability masks. Those masks have the same names as the process masks, and their function is almost the same. The file's inherited mask, however, will limit the capabilities which can be inherited from any other process. David's patch set includes a patch (by Serge Hallyn) which adds support for capability masks to the filesystem layer. 

When a process runs a new executable, the masks are combined as follows: 

  * P′p <- (Pi ∩ Fi) ∪ (Fp ∩ bnd)

  * P′e <- (Pi ∩ Pe ∩ Fi) ∪ (Fp ∩ Fe ∩ bnd)

  * P′i <- P′p

These equations are taken directly from [David's "new capabilities" page](<http://www.madore.org/~david/linux/newcaps/>), which has much more detail on all of this work. What they say, in English, is something like this: 

  * The permitted capabilities for the new executable (P′p) are the intersection of the inheritable set from process before calling `exec()` (Pi) and the file's inherited set (Fi). The permitted set from the file (Fp) is then added in, but not before being limited by the system-wide [capability bounding set](<http://lwn.net/1999/1202/kernel.php3>). 

  * The effective capabilities (P′e) will be the same as the inherited capabilities, except that capabilities which are not effect in the current process or in the file's effective set will be masked out. 

  * The inheritable capabilities (P′i) will be the same as the permitted capabilities. 

For the most part, these rules match the usual understanding of how capability-based systems are supposed to work. Capabilities, in such a system, are assigned to programs, not to users; the normal permissions bits can then come into play to control which programs specific users can run. 

David's patch differs from the usual idea of capability-based systems in one important regard, however: how it handles programs with no capability sets defined. On most systems, that will be almost every executable file there is. By the rules, such programs should be treated as having an empty inherited set, which, by the rules above, would cause them to be run with no capabilities at all. David's patch, instead, causes these programs to be run with the same capabilities the process had before - though the presence of things like setuid bits can obviously change that calculation. This interpretation breaks the classic capability-based model, but it has the advantage of actually working on current systems. 

Ted T'so, however, [complains](<https://lwn.net/Articles/199007/>) that this compromise fundamentally weakens the security of the capability-based model. He has suggested that the behavior be configurable, with each filesystem having a flag describing how capabilities should be handled in the absence of a set per-file masks. A set of default capabilities for new files could be part of this change as well. 

The other complaint which has been heard is fairly predictable: why, it is asked, should we bother with capabilities when SELinux can do all of the same things and more? In fact, SELinux does something vaguely similar, but with a level of indirection; it attaches labels to files, then associates capabilities with the labels through the policy mechanism. Anybody who has ever gotten that cheery Fedora "your filesystem must be relabeled, please wait for a very long time" boot message knows that keeping files and labels properly synchronized is a difficult task. There is no real reason to believe that keeping capability masks in a correct state would be any easier. That fact alone may continue to limit the real usage of capabilities well into the future.
