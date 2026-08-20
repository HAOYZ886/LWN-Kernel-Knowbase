---
title: "The last-minute unshare() discussion"
url: https://lwn.net/Articles/176453/
date: "March 21, 2006"
category: unshare
author: 
---

One of many new system calls added in the 2.6.16 kernel is `unshare()`. Its purpose is to perform the opposite of the various sharing flags provided with `clone()`: it is used to disconnect some of a process's resources from those of its ancestor and sibling processes. With `unshare()`, a process can ask to have its own filesystems, namespaces, or file descriptor table. The unsharing of other resources, including semaphore undo information, virtual memory, signal handlers, and more is stubbed in for future releases. 

A couple of last-second issues with `unshare()` surfaced just as 2.6.16 was being prepared for final release; only some of those issues were resolved in the resulting kernel. 

One of those had to do with the implementation of `unshare(CLONE_VM)`, which causes the calling process to stop sharing memory with others. It seemed that this functionality was present and complete, until Oleg Nesterov noticed that the code does not take into account the possibility that a core dump of the address space may be in process. The solution, for now, is to simply disable unsharing of memory. It seems that there is nobody who needs this feature immediately, and it was too late to be trying to fix up a core memory management function. 

Eric Biederman [raised a couple of other issues](<https://lwn.net/Articles/176456/>) relating to the `unshare()` API which he would have liked to see fixed before that API becomes part of a released kernel. One was the use of the same set of flags used by `clone()` to specify sharing. Eric says: 

sys_unshare can't implement half of the clone flags under any circumstances and those that it does implement have subtlely different semantics than the clone flags. Using a different set of flags sets the expectation that things will be different. 

That discussion did not get very far, however; Linus [prefers to use the same](<https://lwn.net/Articles/176457/>) flags, and nobody else seems to be terribly upset about it. 

Eric's other point was that `unshare()` does not test for unrecognized flags; they are silently ignored. So user space can ask for the unsharing of resources which are not known to - or supported by - the `unshare()` call and no error status will be returned. This behavior could be a problem in the future, when the set of legal flags for `unshare()` is expected to grow. A program written to use one of the new flags may not do the right thing if it is subsequently run on a 2.6.16 kernel; the functionality it asks for will not be present, but the kernel will not inform it of the fact. 

The patch submitted by Eric addressed both issues: the names of the flags and testing for unrecognized flags. It was not merged for 2.6.16, however. The unrecognized flag test, on its own, might have gotten in (and such a patch has been merged for 2.6.17), but the combined patch didn't make it. Andrew Morton [remarked](<https://lwn.net/Articles/176463/>): ""Your single patch did two different things - there's a lesson here"". The creation of tightly-focused patches truly is important, especially just prior to a final kernel release.
