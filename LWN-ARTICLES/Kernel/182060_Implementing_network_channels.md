---
title: Implementing network channels
url: https://lwn.net/Articles/182060/
date: "May 1, 2006"
category: "Kevent; Networking-Channels"
author: 
---

Last January, Van Jacobson [presented his network channel concept](<http://lwn.net/Articles/169961/>) at the 2006 linux.conf.au gathering. Channels, by concentrating network processing in ways which are most friendly to SMP systems, look like a promising way to improve high-speed networking performance. There was a fair amount of excitement about the idea. Unfortunately, Mr. Jacobson appears to have since become busy with other projects, so no contributions of actual code have resulted from his work. So not much has happened on this front in the last few months - or so it seemed. 

David Miller recently [let slip](<https://lwn.net/Articles/182061/>) that he was working on his own channel implementation. It was not something he expected to see functioning anytime soon, however: 

[D]on't expect major progress and don't expect anything beyond a simple channel to softint packet processing on receive any time soon. 

Going all the way to the socket is a large endeavor and will require a lot of restructuring to do it right, so expect this to take on the order of months. 

It turns out, however, that David was not the only person working on this idea; Kelly Daly and Rusty Russell have also put together [a rudimentary channel implementation](<https://lwn.net/Articles/181458/>); in response to David's note, they posted their code for review. Since this version is more advanced, it has been the center of most of the discussion. 

The Daly/Russell patch creates a data structure called `struct channel_ring`. It consists of 256 pages of memory, mapped contiguously into the receiving process's address space - though the pages will not be contiguous in kernel space. As Van Jacobson described, the variables used by the producer side are located at the beginning of the ring, while variables used by the consumer are at the end; this separation helps to ensure that the cache lines representing those variables do not bounce between processors. These variables include the circular buffer indexes indicating which buffer each side will use next. There are also flags allowing the consumer to request a wakeup when buffers are added to the ring. 

User-space starts by creating a socket with the new `PF_VJCHAN` protocol type, then using `mmap()` to map the ring buffer. Thereafter, it can use buffers as they become available (using `poll()` or `select()`, if need be, to wait for more data). When a buffer is no longer needed, incrementing the appropriate index will free it up for new data. 

The driver-side interface is, so far, quite simple. A buffer can be allocated from a given ring with a call to `vj_get_buffer()`; once the data has been placed there by the network interface, `vj_netif_rx()` sends that buffer up into the protocol code. The tricky part is getting each packet into the correct buffer in the first place. Copying packets inside the kernel would defeat the purpose of this whole exercise; it is important that the network interface choose the correct buffer before DMAing the packet data into memory. As it happens, contemporary network cards can be smart enough to make that decision, if programmed properly by the driver. 

There are vast numbers of issues to be worked out still. David Miller [takes exception](<https://lwn.net/Articles/182068/>) to the preallocated buffers, seeing them as inflexible and hard to change; he would rather see a pointer-oriented data structure. But it is hard to see how that might work while still avoiding the overhead of mapping buffers into user space with every packet. 

A more difficult issue, perhaps, is netfilter. The zero-copy approach can be quite fast, but it also naturally shorts out the packet filtering done by the netfilter code. It has been suggested that, for established connections, that is an acceptable tradeoff. But Rusty has [pointed out](<https://lwn.net/Articles/182069/>) that people do use filtering on established connections, for packet counting if nothing else. As he put it: ""Basically I don't think we can 'relax' our firewall implementation and retain trust"". So some other sort of solution will have to be found here. 

Another open issue has to do with whether the channel should go all the way through to user space or not. Van Jacobson's linux.conf.au presentation included discussion of a user-space TCP implementation, taking the end-to-end principle to its logical conclusion. The reasoning behind this move is that, since the data will be processed by the application, putting the protocol code in the same place will be the fastest, most cache-friendly way to do it. But moving protocol code to user space also means duplicating much of the networking stack and adding to the complexity of the system as a whole. Leaving the protocol code in the kernel simplifies the situation, and, it is believed, can be made to yield almost all of the same performance benefits. In particular, protocol processing can happen on the same processor as the destination application (a fair amount of it is done that way now), and zero-copy networking will still be possible. 

It has also been pointed out that, since most of the system calls involved with network data reception (`read()` or `recv()`, for example) already imply copying the data, that copy might as well be done in kernel space. But implicit in that statement is another conclusion: if channels are to be used to their fullest potential for high-performance networking, a new set of user-space interfaces will have to be developed. The venerable socket interface was never designed for a channel-oriented environment. How such an interface might look is not entirely clear; it could be based on the current asynchronous I/O API, on [kevents](<http://lwn.net/Articles/172844/>), or on something completely new. 

In summary, the networking developers are working on some major changes to how networking will be done in Linux, and there are a lot of issues which are not yet understood. The developers are groping around for ideas. So the channel implementations which are being posted now are unlikely to resemble the code which will, someday, be merged into the mainline; they are, instead, exercises intended mainly to obtain a better understanding of the real nature of the problem. But they are still a promising start to what looks to be an interesting development effort.
