---
title: "Kernel Summit 2005: RAS tools"
url: https://lwn.net/Articles/144277/
date: "July 20, 2005"
category: Kexec; RAS tools
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
Suparna Bhattacharya led a session on RAS (reliability, serviceability, and availability) tools. The state of the art has advanced somewhat in the last year; this session was thus mostly a status report, rather than a place where future work was to be discussed. 

The kexec and kdump patches (last [covered here](<http://lwn.net/Articles/140975/>) in June) have been merged into the mainline. Together these patches enable the creation of a far more reliable crash dump capability than Linux has had in the past. There is still work to be done, however, much of it in user space, to get crash dumps to a point where they can be deployed by vendors. 

There's also a few remaining issues. Driver initialization is one of them; after a kernel crash (or any other invocation of a new kernel with kexec) the BIOS initialization will not have been performed. So drivers will have to reset their hardware from an unknown initial state. Getting the frame buffer back into working condition is a challenge in the best of times, and will be made more difficult in a panic situation. It is also important to put an end to any DMA operations which may have been happening when the kernel crash took place. That, in turn, may require a big bus reset, something the kernel normally tries to avoid doing. All of this implies that the kernel needs a flag saying "this is a crash dump kernel" so that it can take the appropriate steps. 

Keeping the analysis tools in sync with the kernel will also be a challenge; the high rate of change affects these tools just as much as it affects drivers. Crash dumps are most likely to be used with "enterprise" distribution kernels, however, which do not change very often. 

There was some talk of relayfs, a tool for getting large amounts of trace data out of the kernel in a hurry. It turns out that the current relayfs implementation allows the trace data to be read via `mmap()`, but not via a normal `read()` call. There was a strange discussion on whether it was appropriate to implement `read()` until Linus decreed that it was silly not to.
