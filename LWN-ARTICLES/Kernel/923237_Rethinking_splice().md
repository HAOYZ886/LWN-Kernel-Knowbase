---
title: Rethinking splice()
url: https://lwn.net/Articles/923237/
date: "February 17, 2023"
category: io uring; splice
author: "By Jonathan Corbet February 17, 2023"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 17, 2023

The [`splice()` system call](<https://man7.org/linux/man-pages/man2/splice.2.html>) is built on an appealing idea: connect two file descriptors together so that data can be moved from one to the other without passing through user space and, preferably, without being copied in the kernel. `splice()` has enabled some significant performance optimizations over the years, but it has also proved difficult to work with and occasionally surprising. A recent linux-kernel discussion showed how `splice()` can cause trouble, to the point that some developers now wonder if adding it was a good idea. 

Stefan Metzmacher is a [Samba](<https://www.samba.org/>) developer who would like to use `splice()` to implement zero-copy I/O in the Samba server. He has [run into a problem](<https://lwn.net/ml/linux-kernel/0cfd9f02-dea7-90e2-e932-c8129b6013c7@samba.org/>), though. If a file is being sent to a remote client over the network, `splice()` can be used to feed the file data into a socket; the network layer will read that data directly out of the page cache without needing to make a copy in the kernel — exactly the desired result. But if the file is written before network transmission is complete, the newly written data may be sent, even though that write happened after the `splice()` call was made, perhaps even in the same process. That can lead to unpleasant surprises (and unhappy Samba users) when the data received at the remote end is not what is expected. 

The problem here is a bit more subtle than it might seem at a first glance. To begin with, it is not possible to splice a file directly into a network socket; `splice()` requires that at least one of the file descriptors given to it is a pipe. So the actual sequence of operations is to splice the file into a pipe, then to connect the pipe to the socket with a second `splice()` call. Neither `splice()` call knows when the data it passes through has reached its final destination; the network layer may still be working with the file data even after both `splice()` calls have completed. There is no easy way to know that the data has been transmitted and that it is safe to modify the file again. 

In his initial email, Metzmacher asked whether it would be possible to prevent this problem by marking file-cache pages as copy-on-write when they are passed to `splice()`. Then, if the file were written while the transfer was underway, that transfer could continue to read from the older data while the write to the file proceeded independently. Linus Torvalds quickly [rejected](<https://lwn.net/ml/linux-kernel/CAHk-=wj8rthcQ9gQbvkMzeFt0iymq+CuOzmidx3Pm29Lg+W0gg@mail.gmail.com/>) that idea, saying that the sharing of the buffers holding the data is ""the whole point of splice"". Making those pages copy-on-write would break sharing of data in general. He later [added](<https://lwn.net/ml/linux-kernel/CAHk-=wj66F6CdJUAAjqigXMBy7gHquFMzPNAwKCgkrb2mF6U7w@mail.gmail.com/>) that a `splice()` call should be seen as a form of [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>), with similar semantics. 

He also said: ""You can say 'I don't like splice()'. That's fine. I used to think splice was a really cool concept, but I kind of hate it these days. Not liking splice() makes a ton of sense."" Like it or not, though, the current behavior of `splice()` cannot change, since that would break existing applications; even Torvalds's dislike cannot overcome that. 

Samba developer Jeremy Allison [suggested](<https://lwn.net/ml/linux-kernel/Y+aKuC1PuvX4STEI@jeremy-acer/>) that the solution to Metzmacher's problem could be for Samba to only attempt zero-copy I/O when the client holds a lease on the file in question, thus ensuring that there should be no concurrent access. He later had to [backtrack](<https://lwn.net/ml/linux-kernel/Y+aat8sggTtgff+A@jeremy-acer/>) on that idea, though; since the Samba server cannot know when network transmission is complete, the possibility for surprises still exists even in the presence of a lease. Thus, he concluded, ""`splice()` is unusable for Samba even in the leased file case"". 

Dave Chinner [observed](<https://lwn.net/ml/linux-kernel/20230210040626.GB2825702@dread.disaster.area/>) that this problem resembles those that have previously been solved in the filesystem layer. There are many cases, including RAID 5 or data compressed by the filesystem, where data to be written must be held stable for the duration of the operation; this is the whole [stable-pages problem](<https://lwn.net/Articles/442355/>) that was confronted almost twelve years ago. Perhaps a similar solution could be implemented here, he said, where attempts to write to pages currently being used in a `splice()` chain would simply block until the operation has completed. 

Both [Torvalds](<https://lwn.net/ml/linux-kernel/CAHk-=wip9xx367bfCV8xaF9Oaw4DZ6edF9Ojv10XoxJ-iUBwhA@mail.gmail.com/>) and [Matthew Wilcox](<https://lwn.net/ml/linux-kernel/Y+XLuYh+kC+4wTRi@casper.infradead.org/>) pointed out the flaw with this idea: the `splice()` operation can take an unbounded amount of time, so it could be used (accidentally or otherwise) to block access to a file indefinitely. That idea did not go far. 

Andy Lutomirski [argued](<https://lwn.net/ml/linux-kernel/CALCETrU-9Wcb_zCsVWr24V=uCA0+c6x359UkJBOBgkbq+UHAMA@mail.gmail.com/>) that `splice()` is the wrong interface for what applications want to do; `splice()` has no way of usefully communicating status information back to the caller. Instead, he said, [io_uring](<https://lwn.net/Articles/776703/>) might be a better way to implement this functionality. It allows multiple operations to be queued efficiently and, crucially, it has the completion mechanism that can let user space know when a given buffer is no longer in use. Jens Axboe, the maintainer of io_uring, was initially [unsure](<https://lwn.net/ml/linux-kernel/7a2e5b7f-c213-09ff-ef35-d6c2967b31a7@kernel.dk/>) about this idea, but [warmed to it](<https://lwn.net/ml/linux-kernel/b44783e6-3da2-85dd-a482-5d9aeb018e9c@kernel.dk/>) after Lutomirski [suggested](<https://lwn.net/ml/linux-kernel/CALCETrVx4cj7KrhaevtFN19rf=A6kauFTr7UPzQVage0MsBLrg@mail.gmail.com/>) that the problem could be simplified by taking the pipes out of the picture and allowing one non-pipe file descriptor to be connected directly to another. The pipes, Axboe said, ""do get in the way"" sometimes. 

Axboe thought that a new "send file" io_uring operation could be a good solution to this problem; it could be designed from the beginning with asynchronous operation in mind and without the use of pipes. So that may be the solution that comes out of this discussion — though somebody would, naturally, actually have to implement it first. 

There was some talk about whether `splice()` should be deprecated; Torvalds [ doesn't think](<https://lwn.net/ml/linux-kernel/CAHk-=wjQZWMeQ9OgXDNepf+TLijqj0Lm0dXWwWzDcbz6o7yy_g@mail.gmail.com/>) the system call has much value: 

> The same way "everything is a pipeline of processes" is very much historical Unix and very useful for shell scripting, but isn't actually then normally very useful for larger problems, splice() really never lived up to that conceptual issue, and it's just really really nasty in practice. 
> 
> But we're stuck with it. 

There is little point in discouraging use of `splice()`, though, if the kernel lacks a better alternative; Torvalds expressed doubt that the io_uring approach would turn out to be better in the end. The only way to find out is probably to try it and see how well it actually works. Until that happens, `splice()` will be the best that the kernel has to offer, its faults notwithstanding.
