---
title: "Understanding the Linux Kernel, 3rd Edition"
url: https://lwn.net/Articles/161190/
date: "November 22, 2005"
category: Books
author: 
---

[![\[UTLK cover\]](https://static.lwn.net/images/ns/kernel/utlk-cover.gif)](<http://www.oreilly.com/catalog/understandlk/>) It's [official](<https://lwn.net/Articles/161192/>): the third edition of _Understanding the Linux Kernel_ , by Daniel P. Bovet and Marco Cesati, is out. Your editor was pleased to receive a copy of this 900-page monster, delivered by a company which specializes in other sorts of heavy loads, such as pianos. UTLK 3 was some time in coming, but it is a welcome arrival. 

As one would expect, this version of UTLK covers the 2.6 kernel. Your editor would like to point out to kernel-oriented publishers, however, that simply saying "2.6" is not particularly informative. A wide variety of kernels have come out under the 2.6 name. Readers will want to know _which_ 2.6 kernel is covered by a given book, and they would rather not have to dig for that information. As it turns out, the reader who gets far enough into the introduction will discover that UTLK 3 was written for the 2.6.11 kernel. 

[![\[UTLK figure\]](https://static.lwn.net/images/ns/kernel/utlk-figure-sm.png)](<https://lwn.net/Articles/161199/>) This version of the book, like its predecessor, is big on details, and is very firmly tied to the i386 architecture. So, after a brief (and, in your editor's opinion, not entirely necessary) introduction to how Unix-like systems work, the book delves directly into i386 segmentation registers, global descriptor tables, and related details. Many of the key control paths in the kernel are described, step by step; often, the book includes flow charts, like the example on the left (which describes the page fault handling algorithm). The level of detail sometimes makes it hard to get a sense for the big picture, but it _does_ help somebody trying to figure out how a particular function works. 

Indeed, that is perhaps the key feature which differentiates this book. It is very much a "how it works" book, designed to help people understand the code. It is not, however, a "how to hack it" book like _Linux Device Drivers_ or _Linux Kernel Development_. It presents kernel functions and data structures, steps the reader through them, but does not, for example, emphasize the rules for using them. UTLK is a study guide, not a programming manual. 

But it is an effective and useful study guide. It covers a wide range of topics, including memory management, process management, scheduling, signals, the virtual filesystem, timing, the I/O layers, and more. Even with its weight, this book cannot cover everything, however; omitted topics include networking, security (security modules, key management, etc.), specific device drivers, sound, video, the kernel build system, and all of the other architectures supported by Linux. (For what it's worth, O'Reilly is said to have an "understanding the Linux network stack" book in the works now). 

Certainly, there are things your editor would have done differently. There are some minor technical glitches; for example, the book claims that acquiring a semaphore always involves putting the acquiring process to sleep first, which is very much not the case. The discussion of sleeping starts with `sleep_on()`, and only later mentions that `sleep_on()` is not a recommended interface. The discussion of some interesting topics (direct I/O, for example) is overly short. But, as a whole, the book is excellent, and the kernel function index at the end helps to make it a useful reference. There is a space on your editor's "L1 bookshelf" (the one reachable without moving the chair) for UTLK 3.
