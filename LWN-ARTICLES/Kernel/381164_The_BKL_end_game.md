---
title: The BKL end game
url: https://lwn.net/Articles/381164/
date: "March 30, 2010"
category: Big kernel lock; lock kernel
author: "By Jonathan Corbet March 30, 2010"
---

By **Jonathan Corbet**  
March 30, 2010

The removal of the big kernel lock (BKL) has been a kernel development goal for many years. The BKL creates scalability problems and provides some truly strange locking semantics that would be nice to eliminate. The actual work of removing this lock has been a long process, though; it is a tedious job requiring a fairly deep understanding of the affected code. Relatively few people are willing to do that work, so the BKL has survived for far longer than anybody might have liked. 

One developer who has put some significant time into BKL removal is Arnd Bergmann; Arnd has just posted [a patch series](<http://lwn.net/Articles/380174/>) which promises to eliminate the BKL altogether - almost. 

To that end, a number of significant changes have been made. The block and tty subsystems both get subsystem-level mutexes to replace their use of the BKL; that is a relatively tricky job because the locking semantics provided by a mutex are rather different. An extensive effort has been made to audit and document `ioctl()` and `llseek()` functions which still require the BKL; no other function called from the `file_operations` structure expects the BKL now. Code still requiring the BKL is now explicitly marked in the kernel configuration system, making it possible to build BKL-free kernels. The patch set also includes a significant series from Jan Blunck removing the BKL from much of the VFS layer. 

What's left is a few ""mostly obscure device driver modules"". Arnd has used a fairly large value of "mostly obscure," though; the USB subsystem, for example, still has a BKL dependency. All told, there are [148 modules](<https://lwn.net/Articles/381166/>) still using the BKL, most of which are drivers. That may seem like a lot, but it's a huge step in the right direction. Many of us may be running BKL-free kernels sooner than we might have expected.
