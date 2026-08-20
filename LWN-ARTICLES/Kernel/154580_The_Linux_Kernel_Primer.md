---
title: The Linux Kernel Primer
url: https://lwn.net/Articles/154580/
date: "October 5, 2005"
category: Books
author: 
---

Your editor recently received a copy of _The Linux Kernel Primer_ , by Claudia Salzberg Rodriguez, Gordon Fischer, and Steven Smolski, published by Prentice Hall. This volume describes itself as "the definitive guide to Linux kernel programming"; it has chapters on processes, scheduling, I/O, filesystems, memory management, and the bootstrap process. It appears to be a guide to internal kernel APIs for the 2.6 kernel. 

Reviewing kernel-related books is a difficult task. Your editor could easily be seen as having a conflict of interest in such cases, with any criticism viewed as an attempt to steer purchasers toward his own, possibly competing work. So, in the interests of full disclosure, let it be said: the author of this review is an author of a different, kernel-related book, and anything found here should be viewed with suspicion. 

Because the simple fact is that your editor cannot recommend this book. It shows every sign of having been put together in a hurry, with basic grammatical errors being a frequent occurrence. The material is disorganized, with no clear ordering of concepts. Factual errors are not hard to find. The sample code provided is visibly buggy. 

The book does not say, anywhere, which version of the kernel is covered - something any serious reader will want to know. Various hints through the text suggest that the authors were working from the 2.6.7 kernel at the latest, however, making the book somewhat obsolete before it hits the shelves. The version of `struct file` shown in the book is from 2.6.1; `struct page` comes from 2.6.4. The list of I/O schedulers does not include CFQ - added in 2.6.6. 

The fundamental fault in this book, however, is this: there is no mention, anywhere, of concurrency issues. Even the few pages devoted to interrupts fail to mention race conditions or the primitives used to control interrupt delivery. Spinlocks and semaphores do not merit coverage until page 409 - and, even then, the API for working with them is not discussed. There is no way to write code for the 2.6 kernel without taking concurrency into account. Your editor cannot understand why the authors felt that this topic could be passed over. 

More documentation for the kernel is a good thing. The kernel is a complex program, and kernel hackers can certainly benefit from a variety of views of how the kernel API works. In this case, however, your editor would recommend staying with the other books in this field, including _Linux Kernel Development_ by Robert Love, and _Understanding The Linux Kernel_ by Bovet and Cesati (third edition due in November).
