---
title: "Book review: Linux System Programming"
url: https://lwn.net/Articles/260916/
date: "December 5, 2007"
category: Books
author: "By Jake Edge December 5, 2007"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
December 5, 2007

![](https://static.lwn.net/images/lss.png)

"System programming" is not easily defined, but is typically considered to consist of programming at a lower level than regular application programming. As Robert Love points out in the introductory chapter of _Linux System Programming_ , there is no technical difference between the two - the same system calls are used - it is more of a difference between programs that implement the infrastructure and programs that use it. Programmers faced with either task will find that understanding how to best use the system call interface is very important. Love sets out to provide that understanding in his book. 

The book is organized into ten chapters: an introduction, three on I/O, two on process management, and one on each of file and directory handling, memory management, signals, and time handling. Each chapter does a good job of covering the subject matter at a level that will help programmers make good choices in the various trade-offs available. The main focus of each chapter is the system calls that Linux provides to perform tasks specific to that area. 

The history of each call is described, along with information about which members of the UNIX family make it available, so that the right choices can be made for portability. Also, various historical (perhaps vestigial is more accurate) calls are documented, with readers being warned away from using them. Each call itself is given a treatment similar to a `man` page, but with greater detail. Where the book really shines is in its comparisons of "similar" system calls. 

The trade-offs between using `select()` and `poll()` or the advantages and disadvantages of using `mmap()` vs. traditional file I/O mechanisms are just two of the comparisons presented. For example, after listing five bulleted advantages of `poll()`, `select()` gets its due: 

The `select()` system call does have a few things going for it, though: 

  * `select()` is more portable, as some Unix systems do not support `poll()`.

  * `select()` provides better timeout resolution: down to the microsecond. Both `ppoll()` and `pselect()` theoretically provide nanosecond resolution, but in practice, none of these calls reliably provides even microsecond resolution. 
Superior to both `poll()` and `select()` is the _epoll_ interface, a Linux-specific multiplexing I/O solution that we'll look at in Chapter 4. 

This is the kind of information that only comes with experience; this book will help a programmer get to that point more quickly. Even for experienced programmers, the comparisons will help crystallize some thoughts that have been floating around. It is definitely one of the better features of the book. 

The book is not without its faults, though, especially in the example code. For each system call, a small example of calling it is provided, but the code snippets are simplistic and do not really provide much meat. There are very few code examples that tie together the various concepts. Had Love done that, there might have been complaints about the size of the resulting book, but the benefit to budding system programmers would be huge. 

There are other problems with the book; for instance, the pirate motif in the examples did not seem to provide anything useful. More seriously, some of the major problems faced by system programmers: race conditions, concurrent data access synchronization, reentrant code, etc. were not covered in much detail. These topics are certainly something a system programmer will need to understand, but they will have to be found elsewhere. 

The back cover of the book describes it as ""an insider's guide to writing smarter, faster code"" - it lives up to some of that, but not all. It _is_ a useful book, however, that will find a home on the bookshelf of many Linux programmers. For those who are relatively new to the topic, there will be a wealth of information. But, even for those who are old hands, there will be useful tidbits, system calls that had escaped notice, and lots of reference material.
