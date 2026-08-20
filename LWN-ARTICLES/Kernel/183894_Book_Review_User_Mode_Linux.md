---
title: "Book Review: User Mode Linux"
url: https://lwn.net/Articles/183894/
date: "May 16, 2006"
category: "Books; User-mode Linux"
author: 
---

There are a number of virtualization technologies available for Linux, some of which have gained a lot of headlines in the last year or two. One of the oldest and most interesting, however, maintains a lower profile. [User-mode Linux](<http://user-mode-linux.sourceforge.net/>) (UML), first implemented by Jeff Dike, takes a unique approach to virtualization. A UML kernel runs within a process on a normal Linux host; it is, essentially, a special port of the kernel designed to run within another Linux system. As a result, a UML system looks like a series of ordinary processes on the host; it can be managed (and debugged) like any other process tree. 

![\[Book cover\]](https://static.lwn.net/images/ns/UML-book-cover.jpg) UML can be somewhat intimidating at first. It brings a new set of acronyms and a whole set of complex configuration options. As with many parts of Linux, the documentation available for UML has not always been everything one might want. So the publication of [_User Mode Linux_](<http://www.phptr.com/title/0131865056>), written by the same Jeff Dike, is a welcome event. This book is part of the Bruce Perens Open Source Series, meaning that it will be released under the Open Content License later this year. For now, however, the book must be obtained the old-fashioned way. For those interested in UML, it should be a worthwhile investment. 

The book adopts a tutorial format, starting with an introduction to UML and virtualization in general. It provides a walk through of a simple UML session, then introduces virtual disks and network interfaces. 

The core of the book is a series of chapters on managing UML and connecting it with the host system (and other UML instances). So there is a chapter on filesystem management, including details on how to provide restricted access to filesystems on the host. A detailed chapter on networking has been provided. UML has several possible network transports which can be used to create isolated networks for UML systems or to connect those systems to the wider world; this chapter covers them all and provides guidance on how to choose between them. Then there is a chapter on the management interface to UML. 

The final set of chapters looks at configuring UML for specific tasks. Chapter 11 talks about building UML from source. In your editor's opinion, that chapter comes a little late; everything to that point has simply assumed that UML is already available on the reader's system. Some distributions have UML packages, but others do not. So some early guidance on how to build a UML system and create an initial filesystem for it to boot from would have been nice. The book finishes with some talk of the (ambitious) future plans for UML and a couple of reference sections. 

There is no clear information on just which version of UML is covered - an unfortunate omission. The sample boot output in the introductory chapter shows 2.6.10 and 2.6.11-rc kernels. 

Minor quibbles aside, it is hard to find much to complain about in Jeff's book. It provides a much-needed reference for an important Linux virtualization mechanism. There are a number of possible uses for UML, including kernel development, server consolidation, embedded systems development, experimenting with different distributions, or the simple joy of running a large cluster on one's laptop. Regardless of their goal, UML users will find this book to be a worthwhile addition to their shelves.
