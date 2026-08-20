---
title: "The return of kevent?"
url: https://lwn.net/Articles/233462/
date: "May 8, 2007"
category: Events reporting; Kevent
author: 
---

The last time this page looked at [the kevent interface](<http://lwn.net/Articles/213672/>), it seemed to have reached the end of its run. The [eventfd](<http://lwn.net/Articles/225714/>) patches had stolen the thunder, providing a way for applications to wait on many types of events using the standard polling interfaces. The kevent developer has shelved the work on the assumption that it would not get in. That assumption appeared to be justified, given that Andrew Morton, in his [2.6.22 merge plans document](<http://lwn.net/Articles/232432/>) said that the eventfd patches would be included. 

As was mentioned last week, one obstacle came up in the form of [pollfs](<http://lwn.net/Articles/232685/>), an implementation of a very similar idea. There were a couple of relatively harsh reviews of the pollfs code, and its profile appears to have lowered considerably. It is possible that a new, improved version of pollfs could show up in the near future, but it would have to be a lot better to grab a significant amount of attention. The pollfs code has probably shown up too late to the game. 

There's another late arrival who will have to be listened to, however: glibc maintainer Ulrich Drepper. Having sat out the discussion of eventfd, he is now back and [opposing its inclusion](<https://lwn.net/Articles/233464/>) into the mainline: 

It's Linus decision whether he wants to add yet more code, yet more possible problems, yet more maintenance overhead/nightmare for an interim solution which isn't necessary, which cannot solve all the problems, and which is not as scalable as other proposed methods. 

I can only say that I would be trickly [sic] against it. It makes just no sense. 

Ulrich has a number of complaints about the eventfd approach: 

  * The eventfd code, by relying on `poll()` and variants, does not provide a way for applications to obtain events without entering the kernel. For high-bandwidth applications - big network servers, for example - eliminating system calls is one of the keys to adequate performance. The kevent code, with its user-space event ring, provides that sort of mechanism while eventfd does not. 

  * The use of `poll()` also makes it hard for the kernel to pass information back to the application - the communication channel only includes a few bits. The kevent interface allows for a fair amount of information to be packaged with each event. Eventfd gets around this problem by allowing applications to read more event information from the relevant file descriptors - but that requires another system call. 

  * Ulrich [argues](<https://lwn.net/Articles/233465/>) that the `poll()` interface poses unsolvable issues with regard to threads and cancellation processing. This argument is [not universally accepted](<https://lwn.net/Articles/233466/>), however. 

  * The current eventfd code does not let applications wait on futexes, and Davide Libenzi, the eventfd developer, [is uninclined](<https://lwn.net/Articles/233467/>) to add that support. The pollfs patches do support futex waits, though Ulrich had some issues with the implementation. In general, Ulrich would like to see a single system call where applications can wait for anything, so leaving out primitives like futexes will leave him unsatisfied. 

The end result of this is that Ulrich opposes the merging of eventfd; he would rather see the effort go into making kevent (or a replacement with similar functionality) ready for the mainline. A kevent-like interface, he [says](<https://lwn.net/Articles/233468/>), will eventually become necessary in any case: 

I think we ultimately have to have something like kevent and then all this *fd() work is unnecessary and just adds code to the kernel which has to be kept around and which might hinder further work in this area. 

How this issue will be resolved is entirely unclear. There's not been a flood of developers lining up to support Ulrich's position - but they are not opposing him either. Nobody has dusted off the kevent patches for another round of discussion - yet. But one thing that does seem likely is that this whole discussion may delay the merging of eventfd past the 2.6.22 merge window. User-space interfaces are important and, once they are added to the kernel, they are almost impossible to remove. Waiting another development cycle seems like a small price to pay if it helps the developers to get this decision right. 

**Update** : the eventfd code was merged into the mainline on May 11.
