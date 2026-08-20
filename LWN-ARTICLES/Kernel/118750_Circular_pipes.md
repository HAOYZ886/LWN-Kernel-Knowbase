---
title: Circular pipes
url: https://lwn.net/Articles/118750/
date: "January 11, 2005"
category: Circular buffers; Pipes; splice
author: 
---

One of the many changes slipped quietly into BitKeeper over the last week was [this patch from Linus](<https://lwn.net/Articles/118751/>) changing how pipes are implemented internally. For a long time, pipes have used a single page to buffer data between the reader and the writer. If a process writes more than one page, it will block until the reader has consumed enough data to allow the rest to fit within the buffer. The 2.6.11 pipe implementation will be rather different. 

Pipes now use a circular buffer, as inexpertly shown in the diagram below: 

![\[Circular pipe diagram\]](https://static.lwn.net/images/ns/kernel/circular-pipe.png)

The `curbuf` pointer (it's an integer index, actually) indicates the first buffer in the array which contains data; `nrbufs` tells how many buffers contain data. The `page` structures are allocated when needed, and do not hang around when not in use. Since both readers and writers can manipulate `nrbufs`, some sort of locking (the pipe semaphore, in this case) is needed to serialize access. The `pipe_buffer` structure includes length and offset fields, so each entry in the circular buffer can contain less than a full page of data. 

Linus [says](<https://lwn.net/Articles/118756/>) that the new implementation gives a "30-90%" improvement in pipe bandwidth, with only a small cost in latency (since pages must be allocated when data passes through the pipe). The performance improvements are entirely attributable to the larger amount of buffering; readers and writers will block less often when passing data through the pipe. It is a way of speeding things up by throwing memory at the problem. 

Better pipe performance was not Linus's main purpose in making this change, however; he has a longer-term plan in mind. The mechanism used to implement circular pipes will evolve into a general mechanism for passing data streams through the kernel. Quite a few changes will be required to get there, and there seems to be no hurry, but there is clearly a long-term goal in mind. 

Among other things, the buffers within the circular structure will gain a reference count, allowing there to be multiple readers or writers. The idea here is to implement a sort of in-kernel `tee` operation which would let data streams be split without additional copying. The example given by Linus is some sort of video capture device which would feed its data into one of these buffers. A process could obtain data from the buffer and display it in an on-screen window; meanwhile, another process would be capturing the stream and writing it to a file somewhere - perhaps with little or no user-space intervention. 

The circular buffers will also gain the usual structure full of method pointers which would allow specific users to change how the basic operations are performed. Once that is in place, two new system calls would be added: 

`splice(int infd, int outfd);`
    This call would use a circular buffer to transfer data from `infd` to `outfd`, possibly in a zero-copy manner. 

`tee(int infd, int out1, int out2);`
    Arranges for data from `infd` to go to both `out1` and `out2`

Longtime followers of Linux kernel discussions will notice a strong similarity between all of the above and Larry McVoy's [splice proposal](<http://lwn.net/2001/0125/a/splice.php3>). Linus's implementation [works at a lower level](<https://lwn.net/Articles/118759/>), however, and avoids many of the problems he saw with Larry's approach. Those who are curious about where all this is going may want to look at [this explanation from Linus](<https://lwn.net/Articles/118760/>), where he goes into detail and concludes: 

I'm clearly enamoured with this concept. I think it's one of those few "RightThing(tm)" that doesn't come along all that often. I don't know of anybody else doing this, and I think it's both useful and clever. If you now prove me wrong, I'll hate you forever ;) 

There is a remaining practical issue with the current implementation. No coalescing of data written into a circular buffer is performed. Linus did things that way because he wants to make life easy for high-bandwidth, zero-copy streams using these buffers. To that end, nothing touches a page once it has added to a buffer. The problem is that, in the worst case, a process writing a single byte at a time to a pipe can consume 16 pages of memory (with the default configuration) to hold 16 bytes worth of data. Linus initially noted that nobody doing single-byte I/O should expect good performance, and suggested that people not do that. It turns out, however, that this behavior [breaks a crucial application](<https://lwn.net/Articles/118762/>) \- highly parallel kernel compiles. So coalescing of writes is likely to be added in the near future.
