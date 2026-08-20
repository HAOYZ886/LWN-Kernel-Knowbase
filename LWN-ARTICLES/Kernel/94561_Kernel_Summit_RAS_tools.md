---
title: "Kernel Summit: RAS tools"
url: https://lwn.net/Articles/94561/
date: "July 21, 2004"
category: RAS tools
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
Suparna Bhattacharya led a discussion on RAS tools. She noted that there are too many tools out there, and too much complexity. The current set of tools is too complex, and requires too much ahead-of-time setup work. There is no motivation for users to do this work, so they lack tools like crash dump analyzers when they need them. Something simpler and easier to use is needed. 

At the minimum, what is called for is a way to capture data from the kernel. It should be small, self-sufficient, and be able to work with standard tools. Thus, Suparna advocates using the kexec patch in a mode where it preserves memory from the old system. That memory shows up as `/dev/hmem` under the new kernel, where it can be analyzed at leisure. Taking a crash dump is a matter of a `cp` command; analyzing can be done with `gdb`. Obviously, not all of kernel memory can be preserved in this manner (or the new kernel would have no memory to work with), but even a relatively small slice can be useful. 

Additionally, Suparna would like to see the KProbes patch put into the kernel. It needs in-kernel users to motivate that inclusion, though. The full patch requires putting an interpreter into the kernel, which is likely to be a hard sell. 

Karim Yaghmour got up to talk about the Linux Trace Toolkit. His point is that debugging of timing-related problems can be almost impossible without the right sort of data; LTT can provide that data. When queried about the overlap between LTT and the lightweight audit framework recently merged into the kernel, Karim noted that he was there first: LTT has been available for several years. He has been pushing for inclusion for much of that time; this session may have helped that cause, maybe. 

[>> Next: Networking summit summary](<https://lwn.net/Articles/94565/>).
