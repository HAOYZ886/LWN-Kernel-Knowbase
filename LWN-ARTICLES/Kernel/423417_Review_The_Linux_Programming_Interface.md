---
title: "Review: The Linux Programming Interface"
url: https://lwn.net/Articles/423417/
date: "January 19, 2011"
category: Books
author: "By Jake Edge January 19, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
January 19, 2011

Michael Kerrisk's (relatively) new book, _The Linux Programming Interface_ (TLPI), is targeted at Linux system programmers, but it is not just those folks who will find it useful. While it is a hefty tome (""thick enough to stun an ox"" as Laurie Anderson might say), it is eminently readable, both by browsing through it or by biting the bullet and reading it straight through. The coverage of the Linux system call interface is encyclopedic, but the writing style is very approachable. It is, in short, an excellent reference that will likely find its way onto the bookshelves of user-space developers and kernel hackers--including some who aren't necessarily primarily focused on Linux. 

![\[TLPI cover\]](https://static.lwn.net/images/2011/TLPI-front-cover.png)

Kerrisk has been the maintainer of the [Linux man pages](<http://kernel.org/doc/man-pages/>) since 2004, which gives him a good perspective on the Linux API. As he says in the preface, it is quite likely that you have already read some of his work in sections 2, 3, 4, 5, and 7 of those pages. But the book is not a collection of man pages though it covers much of the same ground. The style and organization is much less dry, and more explanatory, than a typical man entry. 

The book is some 1500 pages in length, which makes it a rather daunting prospect to review. Once I started reading it, though, it was quite approachable. Kerrisk's clear descriptions of various system calls and other parts of the Linux API made it easy to keep reading. I set out to pick and choose certain chapters to read, and just skim the others, but found myself reading quite a bit more than that--which might partially explain the lateness of this review. 

The book is organized into 64 chapters of around 20 pages each, which makes for nice bite-sized chunks that allow for reading the book around other tasks. While the focus is on Linux, Kerrisk doesn't neglect other Unix varieties and notes where they differ from Linux. He also pays careful attention to the various standards that specify Unix behavior--like POSIX and the Single Unix Specification (SUS)--pointing out where Linux does and does not follow those standards. 

TLPI was written for kernel version 2.6.35 and glibc 2.12. In the text, though, Kerrisk is careful to indicate which kernel version introduced a new feature, so that those working with older kernels will know which they can use. While it is primarily looking at the 2.6 series, 2.4 is not neglected, and the text notes features that were introduced at various points in the 2.4 kernel history. 

The book starts with a bit of history, going all the way back to Ken Thompson and Dennis Ritchie and then moving forward to the present, looking at the various branches of the Unix tree. It then moves into a description of what an operating system is, the role that the kernel plays, and some of the overarching concepts that make up Unix (and Linux). While this information may be unnecessary for most Linux hackers, it will come in handy for those coming to Linux from other operating systems. The ideas that "everything is a file" and that files are just streams of bytes are described in ways that will quickly get a system programmer up to speed on the "Unix way". 

After that introductory material, Kerrisk launches into the chapters that cover aspects of the system call interface. This makes up the vast majority of the book and each of these chapters is fairly self-contained. They build on the earlier chapters, but the text is replete with references to other sections. In the preface, Kerrisk says that he attempted to minimize forward references, but that clearly was a difficult task as there are often as many forward as backward references in a chapter. 

Navigating within the book is easy to do because there are frequent numbered section and subsection headings, along with the chapter number on each page. Other technical books could benefit from that style. There is also an almost too detailed index that runs to more than 50 pages. 

Each chapter comes with sample code that is easy to read and understand. Importantly, the examples also do a good job of demonstrating the topic at hand and some of them could be adapted into useful utilities. The code is available from the [TLPI web site](<http://man7.org/tlpi/>) and is free software released under the Affero GPLv3. Each chapter also has a handful of exercises for the reader, some of which have answers in one of the appendices. 

So, what does the book cover? It would be easy to say "all of it", but that would be something of a cop-out, and a bit inaccurate as well. There are multiple chapters on files, file I/O, filesystems, and file attributes, extended attributes, and access control lists (ACLs). There is a chapter covering directories and links, as well as one that looks at the inotify file event notification call. 

There are multiple chapters on processes, threads, signals, as well as chapters covering process groups and sessions, and process priorities and scheduling. Of particular interest to me were a chapter on writing secure privileged programs and one on Linux capabilities. There are two chapters on shared libraries, the first of which is more about the ideas underlying libraries and shared libraries along with how to build them, rather than the `dlopen()` system call (and friends), which is covered in the second. 

There are, perhaps, too many chapters covering interprocess communication (IPC), with separate chapters devoted to each System V IPC mechanism (shared memory, message queues, and semaphores). There is also a chapter for each of the POSIX variants of those three IPC types. Both POSIX and System V IPC get their own introductory chapter in addition to the chapters focusing on the details of each type. Sandwiched between the System V and POSIX IPC mechanisms are two chapters on memory mapping and virtual memory operations that might have been better placed elsewhere in the book. There is also a chapter devoted to an introduction to IPC and one that looks at the more traditional Unix pipes and FIFOs. In all, there are twelve chapters on IPC before we even get to the sockets API. 

After IPC, comes a chapter on file locking followed by six chapters covering sockets. Those chapters look at Unix and internet domain sockets, along with server design and advanced sockets topics. The book wraps up with a chapter on each of terminals and pseudoterminals, with something of an oddly placed "Alternative I/O Models" chapter in between them. It's an interesting chapter, covering `select()`, `poll()`, `epoll()`, signal-driven I/O, and a few other topics, but it seems weird where it is. 

There is more, of course, and looking at the [detailed table of contents](<http://man7.org/tlpi/toc-detailed.html>) will fill out the list. One thing that stands out from the book is the vast size of the Linux/Unix API. It also points out some of the warts and historical cruft that is carried along in that API. Kerrisk is not shy about noting things like that where appropriate in the text: ""In summary, System V message queues are often best avoided."" 

There were two specific topics that I looked forward to reading about but were only marginally covered by the book. The first is containers and namespaces, which are very briefly mentioned in a discussion of the flags to the `clone()` system call. A more puzzling omission is that there is almost no mention of the `ptrace()` system call. In the few places it does come up, readers are referred to the `ptrace(2)` man page. 

There are certainly other parts of the Linux API that could have been covered, beyond the system call interface--sysfs, `splice()`, and perf come to mind--but Kerrisk undoubtedly needed to draw the line somewhere. Overall, he did an excellent job of that. Technical books, especially those covering Linux, have a tendency to get stale rather quickly, but TLPI shouldn't suffer from that as much as a kernel internals book would, for example. There should really only be additions down the road as the user-space API is maintained by the kernel developers "forever", but updates will presumably need to be made eventually. 

There are a handful of additional complaints I could make about the book, but they are all quite minor, as were those mentioned above. The biggest nit is that the "asides" in the text, which are numerous, are really often much more than just asides. Each is set off from the rest of text, indented and rendered in a slightly smaller font (which is typographically a bit annoying to me), and are meant to contain additional information that is not necessarily critical to understanding the topic. In my experience, though, many of them might best have been worked into the main text. See what I mean about minor complaints? 

This is a book that will be useful to application and system-level developers, primarily, but there is much of interest for others as well. Kernel hackers will find it useful to ensure their new feature (or fix) doesn't break the existing API. Programmers who are primarily targeting other Unix systems may also find it useful for making their code more portable. I found it to be extremely useful and expect to return to it frequently. Anyone who has an interest in programming for Linux will likely feel the same way.
