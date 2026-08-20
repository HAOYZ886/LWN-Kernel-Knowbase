---
title: A pair of Rust kernel modules
url: https://lwn.net/Articles/907685/
date: "September 12, 2022"
category: "Development tools-Rust"
author: "By Jonathan Corbet September 12, 2022 Kangrejos"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 12, 2022

* * *

[Kangrejos](<https://lwn.net/Archives/ConferenceByYear/#2022-Kangrejos>)

The idea of being able to write kernel code in the Rust language has a certain appeal, but it is hard to judge how well that would actually work in the absence of examples to look at. Those examples, especially for modules beyond the "hello world" level of complexity, have been somewhat scarce, but that is beginning to change. At the 2022 Kangrejos gathering in Oviedo, Spain, two developers presented the modules they have developed and some lessons that have been learned from this exercise. 

#### An NVMe driver

Andreas Hindborg was up first to talk about [an NVM Express driver](<https://github.com/metaspace/rust-linux/commit/3446d310d9082c00eecb4b396a167e3f19b3b2f1>) written in Rust. The primary reason for this project, he said, was to take advantage of the memory-safety guarantees that Rust offers and to gain some real-world experience with the language. His conclusions from this project include that Rust comes with a lot of nice tooling and that its type system is helpful for writing correct code. It is, he said, easier to write a kernel driver in Rust than in C. 

[![\[Andreas
Hindborg\]](https://static.lwn.net/images/conf/2022/kang/AndreasHindborg-sm.png)](<https://lwn.net/Articles/907716/>) Why write an NVMe driver when the kernel already has one that works well? There are no problems with the existing driver, he said, but NVMe is a good target for experiments with driver abstractions. NVMe itself is relatively simple, but it has high performance requirements. It is widely deployed, and the existing driver provides a mature reference implementation to compare against. 

Hindborg talked for a while about the internals of the NVMe interface; in short, communications between the interface and the computer go through a set of queues. Often the driver will configure an I/O queue for each core in the system if the interface can handle it. Creating data structures in Rust to model these queues is a relatively straightforward task. In the end, the Rust driver, when tested with the [FIO](<https://fio.readthedocs.io/en/latest/fio_doc.html#>) tool, performs _almost_ as well as the existing C driver. The difference, Hindborg said, is that the C driver has already been highly tuned, while the Rust driver has not; it should be able to get to the same level of performance eventually. 

He concluded by saying that the Rust NVMe driver is still "a playground" and not production-ready at this point. To move things forward, he would like to create more abstractions that would allow the removal of the remaining `unsafe` blocks in the driver. It doesn't yet support device removal or the sysfs knobs for the `nvme-cli` tool. He would also like to look into using the Rust async model, which would "simplify a lot of things" in the driver, but possibly at the cost of performance. 

At the end of Hindborg's talk, Paul McKenney asked if there was any available information on the relative bug rates between the C and Rust drivers. Hindborg answered that there have certainly been some bugs; building Rust abstractions around existing C code can be hard to do correctly. That work needs a lot of care and review, but once it works, drivers built on it tend to show few problems. 

#### A 9P filesystem server

Last year, Linus Walleij [suggested](<https://lwn.net/ml/linux-kernel/CACRpkdat-4BbKHMBerdxXBseMb9O3PiDRZmMLP_OWFE2ctSgEg@mail.gmail.com/>) that, rather than writing drivers in Rust, developers should target areas with a higher attack surface — network protocols, for example. Wedson Almeida Filho has taken that advice and written [an in-kernel server](<https://github.com/wedsonaf/linux/commits/9p>) for the [9P](<https://en.wikipedia.org/wiki/9P_\(protocol\)>) filesystem protocol in the hopes that this project would demonstrate the productivity gains and security benefits that Rust can provide. Initially, he had started trying to replace the `ksmbd` server, but that turned out to not be an ideal project. The SMB protocol is too complex and the server needs some significant user-space components to work. He wanted something simpler; 9P fit the bill. 

[![\[Wedson Almeida
Filho\]](https://static.lwn.net/images/conf/2022/kang/WedsonAlmeidaFilho-sm.png)](<https://lwn.net/Articles/907717/>) The 9P file protocol, he said, comes from the [Plan 9](<https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs>) operating system. The kernel has a 9P client, but no 9P server. There is a 9P server in [QEMU ](<https://www.qemu.org>) that can be used to export host filesystems into a guest. The protocol is simple, Almeida said, defining a set of only ten operations. His 9P server implementation works now, in a read-only mode, and required just over 1,000 lines of code. 

Almeida was also looking for a way to experiment with [async Rust](<https://rust-lang.github.io/async-book/>) in the kernel. In the async model, the compiler takes thread-like code and turns it into a state machine that can be implemented with "executors" and "reactors", which are implemented in the kernel crate. He created an executor that can run async code in a kernel workqueue; anywhere such code would block, it will release the workqueue thread for another task. There is also a socket reactor that is called for socket-state changes; it will call `Waker::wake()` from the Rust kernel crate to get the appropriate executor going again. 

There is, of course, plenty of work yet to be done. He would like to implement reactors for other I/O submission paths, including KIOCBs (asynchronous I/O), URBs (USB devices), and BIOs (block devices). Memory allocation can still use some work; it would be good if a `GFP_KERNEL` could give up its thread while waiting for the memory-management subsystem to do complicated things. 

At the end, I asked whether the objective of demonstrating the security benefits of Rust had been achieved; has there been, for example, any fuzz testing of the server? Almeida answered that the Rust-based parsing interface makes a lot of mistakes impossible. No fuzz testing has been done — the server has only been working for a couple of weeks — but he will do it. He concluded that he will be interested to see how his server fares in such testing relative to the QEMU implementation. 

[Thanks to LWN subscribers for supporting my travel to this event.]
