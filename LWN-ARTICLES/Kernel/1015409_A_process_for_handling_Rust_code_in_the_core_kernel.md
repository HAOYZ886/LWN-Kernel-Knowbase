---
title: A process for handling Rust code in the core kernel
url: https://lwn.net/Articles/1015409/
date: "March 27, 2025"
category: "Development tools-Rust"
author: "By Jonathan Corbet March 27, 2025 LSFMM+BPF"
---

By **Jonathan Corbet**  
March 27, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The 2024 Linux Storage, Filesystem, Memory-Management, and BPF Summit included [a tense session](<https://lwn.net/Articles/978738/>) on the use of Rust code in the kernel's filesystem layer. The Rust topic returned in 2025 in a session run by Andreas Hindborg, with a scope that also covered the storage and memory-management layers. A lot of progress has been made, and the discussion was less adversarial this year, but there are still process issues that need to be worked out. 

#### Getting Rust code upstream

There are, Hindborg began, several paths by which Rust code gets into the kernel, which he arranged on ""the Brauner-Hellwig scale"". At the Brauner end, some subsystems are accepting Rust code directly and maintaining it themselves; the virtual filesystem layer (co-maintained by Christian Brauner) is a case in point. In these subsystems, Rust patches are simply accepted like any others. 

[![\[Andreas
Hindborg\]](https://static.lwn.net/images/conf/2025/lsfmm/AndreasHindborg-sm.png)](<https://lwn.net/Articles/1015411>) Moving down the scale, the developers of some subsystems would like to be able to accept Rust code directly, but the maintainers acknowledge that they need help to do so. The kernel's module loader is an example here; to merge that code, experts from the Rust community have been helping. Other subsystems designate a separate sub-maintainer to handle Rust code; the block layer was mentioned as using this model. 

Then, there are the subsystems that don't handle Rust code at all; for these, code goes through the Rust trees instead. In some cases, the maintainers are willing, and will act as reviewers, but don't feel that they can handle the code directly; timekeeping was mentioned here. Finally, some subsystems are not working with the Rust developers in any way; they simply don't want to know about the Rust code. [That is the case for the DMA-mapping](<https://lwn.net/Articles/1006805/>) and XArray subsystems. 

None of these models are wrong, Hindborg said, but the last mentioned is the least productive of all of them. A subsystem's C maintainers know the subsystem best, so their input is needed to get the best results. He said that an understanding of Rust is not necessary to review submissions; the Rust code is extensively documented, and comments on the intended semantics of that code are the most important. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Ted Ts'o said that it would help to set more explicit expectations around testing and breakage. If Rust code is going through the regular subsystem trees, the maintainers should at least be doing build tests of that code, and there should be a defined path forward when a build break happens. If the tree doesn't build, he asked, should patches just be dropped after a certain time? Hindborg answered that, if the break is observed before merging, the patches should just be dropped right away. But mostly, he said, when a break happens, people just fix it and move on. 

David Hildenbrand noted that changing a C interface could make it necessary to change a lot of Rust code to match. If the Rust code is not maintained in the same tree, it is not clear how the patches should be split. Hindborg said that such patches can be handled like any tree-wide series. If need be, the developer can create a new API and slowly convert code over to it. If a developer is having trouble making changes to Rust code, they can ask for help. 

Liam Howlett said that, if he merges a change that breaks Rust code, it will cause immediate problems, such as breaking continuous-integration builds. Then the finger of blame will point at him. It is going to be important to be able to do all of these changes together. Hindborg agreed, but also pointed out that, in the block subsystem, developers make changes without worrying about the Rust side. When he gets a problem report, he fixes it within a day and all is well. Howlett answered that this approach will not scale; more subsystems using Rust will lead to more troubles. Hindborg agreed that it would be better to get all of the changes in a single series, but he didn't think he could ask that of the development community — or get it if he did ask. Petr Tesařík pointed out that breaking the Rust code could lead to trouble for developers who are trying to bisect regressions. 

Hindborg asked block-subsystem maintainer Jens Axboe about his experience with accepting Rust code; Axboe answered that it has added almost no overhead at all. If something breaks, he said, he often gets a patch fixing the problem before he is even aware of it. It is all working well, but he pointed out that the block subsystem does not yet have a lot of Rust code. When asked by Hindborg, Axboe said that he does his builds with Rust enabled, but is not able to run-test everything. 

Ts'o said that he had not been doing Rust-enabled builds until recently, and still does not do them regularly. The reason is that his test infrastructure is based on Debian stable, and that distribution does not have a sufficiently recent version of the Rust compiler. He has gotten things working using Debian testing, but running that distribution invites other problems. Hindborg said that this is an easy problem to solve, all that is needed is to wait. The minimum version of Rust has been 1.78.0 for a while now; over time, the problems will go away. 

Matthew Wilcox said he doesn't have current Rust on his systems because he does not like being a system administrator. He leaves that work up to the distributor; until it provides a suitable compiler, he just won't have it. Jason Gunthorpe said that he had tried to do the Rust build on Ubuntu LTS, but that the instructions in [the kernel documentation](<https://docs.kernel.org/rust/>) do not work there; Hindborg said that he would ensure that it was fixed. 

#### Current and future status

A set of abstractions for the configfs filesystem is ready for consideration, Hindborg said, but the maintainer team for configfs has recently been ""cut in half"" (after Christoph Hellwig stopped doing that work). There is only one maintainer, who is not responding to email. It turns out that the listed maintainer has retired; Hindborg was advised to remove him from the maintainer entry and take that subsystem over. 

There have been a number of advances recently, Hindborg said. The "rnull stub" (a block null-device driver) has been merged, he said, as has Rust support for 32-bit x86 systems. It is now possible to download Rust toolchains from kernel.org. Other merged code includes a user-space access module, abstractions for `struct page`, `struct file`, and `struct cred`, and a linked-list implementation. The [rendered internal documentation](<https://rust.docs.kernel.org/kernel/>) is available on kernel.org. 

The first "real" driver in Rust, for Applied Micro QT2025 PHY devices, has been merged. The KASAN sanitizer and Clang control-flow integrity both work with Rust code. There is an abstraction for misc devices, support for tracepoints, and for RCU read locking. The addition of the memory-management abstractions also led to some significant improvements to the virtual memory area (VMA) locking documentation, he said. 

Looking forward, Hindborg said that the controversial DMA-mapping abstractions will be landing soon [that may have happened by the time you read this]. There will be support for high-resolution timers, module parameters, I/O polling, the XArray data structure, the `iov_iter` mechanism, and VMAs. In the longer term, the Nova graphics driver is moving forward; that will be a multi-year effort. 

In his conclusion, he repeated that the minimum version of the Rust compiler has remained at 1.78.0 since the 6.10 release, even as newer versions have gained support. There are only four unstable features needed by the kernel at this point, and that number will soon be zero, he said. The compiler team includes kernel builds in its continuous-integration testing to be sure they don't break it. The kernel, he said, is a "flagship goal" for the Rust project; the Rust developers take the kernel project seriously, want to make it work, and are dedicating resources toward that end.
