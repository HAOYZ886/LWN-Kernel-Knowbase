---
title: "IPC medley: message-queue peeking, io_uring, and bus1"
url: https://lwn.net/Articles/1065490/
date: "April 2, 2026"
category: Message passing; bus1; io uring
author: "By Jonathan Corbet April 2, 2026"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 2, 2026

The kernel provides a number of ways for processes to communicate with each other, but they never quite seem to fit the bill for many users. There are currently a few proposals for interprocess communication (IPC) enhancements circulating on the mailing lists. The most straightforward one adds a new system call for POSIX message queues that enables the addition of new features. For those wanting an entirely new way to do interprocess communication, there is a proposal to add a new subsystem for that purpose to io_uring. Finally, the bus1 proposal has made a return after ten years. 

#### Peeking at message queues

The POSIX message-queue API is not heavily used, but there are users out there who care about how well it works. Message queues are named objects that, by default, all share a global namespace, though IPC namespaces can be used to separate them. There is a whole set of system calls for the creation, configuration, use, and destruction of message queues; see the [mq_overview man page](<https://man7.org/linux/man-pages/man7/mq_overview.7.html>) for an introduction to this subsystem. 

Of interest here is [`mq_timedreceive()`](<https://man7.org/linux/man-pages/man2/mq_timedreceive.2.html>), which can be used to receive messages from a message queue: 

```
ssize_t mq_timedreceive(size_t msg_len;
                                mqd_t mqdes, char  msg_ptr[msg_len],
                                size_t msg_len, unsigned int *msg_prio,
                                const struct timespec  abs_timeout);
```

This call will receive the highest-priority message pending in the queue described by `mqdes` (which is a file descriptor on Linux systems) into the buffer pointed to by `msg_ptr`, which must be at least `msg_len` bytes in length. If `abs_timeout` is not null, it specifies how long the call should block before returning a timeout error. On successful receipt of a message, the location pointed to by `msg_prio` (if non-null) will be set to the priority of the received message. 

That system call has a fair number of parameters, but Mathura Kumar would like to add some more. Since `mq_timedreceive()` was not designed for extensibility, that means adding a new system call. Thus, [Kumar's patch set](<https://lwn.net/ml/all/20260320052340.6696-1-academic1mathura@gmail.com>) adding `mq_timedreceive2()`. But there is an additional constraint here: there are architecture-imposed limits on the number of arguments that can be passed to system calls, and Kumar's plans would exceed those limits. As a result, the new system call is defined as: 

```
struct mq_timedreceive2_args {
            size_t         msg_len;
            unsigned int  *msg_prio;
            char          *msg_ptr;
        };
    
        ssize_t mq_timedreceive2(mqd_t mqdes,
                                 struct mq_timedreceive2_args *uargs,
                                 unsigned int flags,
                                 unsigned long index,
                                 const struct timespec *abs_timeout);
```

The `msg_len`, `msg_prio`, and `msg_ptr` arguments have been moved into the new `mq_timedreceive2_args` structure, freeing up two slots for new parameters to the system call. That structure is passed by pointer, without using the common pattern of passing its length, which would make future additions easier; that may change if this patch series moves forward. 

The new arguments are `flags` and `index`. In this series, only one flag (`MQ_PEEK`) is defined; if it is present, the message will be returned as usual, but without removing it from the queue, meaning that it will still be there the next time a receive operation is performed. The `index` argument indicates which message is of interest; a value of zero will return the highest-priority message, and higher values will return messages further back in the queue. 

There are a few use cases for these features described in the patch cover letter. One would be monitoring tools, which may want to look at the message traffic without interfering with it. Another one is [Checkpoint/Restore in Userspace](<https://criu.org/Main_Page>), which can read a series of messages out of a queue, then restore them with the rest of the process at a future time. 

The series as a whole has not received much attention so far, which is perhaps unsurprising given that few developers have much interest in POSIX message queues. If this work is to proceed, it will need to attract some reviews, and probably go through some more rounds to address the problems that are found. 

#### IPC in io_uring

Since its inception, the [io_uring subsystem](<https://www.man7.org/linux/man-pages/man7/io_uring.7.html>) has steadily gained functionality. After having started as the asynchronous I/O mechanism that Linux has long lacked, it has evolved into a separate system-call interface providing access to increasing amounts of kernel functionality. While io_uring can be used for interprocess communication (by way of Unix-domain sockets, for example), it has not yet acquired its own IPC scheme. [This patch series](<https://lwn.net/ml/all/20260313130739.23265-1-git@danielhodges.dev>) from Daniel Hodges seeks to change that situation, but it probably needs a fair amount of work to get there. 

Hodges's goal is to provide a high-bandwidth IPC mechanism, similar to [D-Bus](<https://www.freedesktop.org/wiki/Software/dbus/>), that will perform well on large systems. By using shared ring buffers, processes should be able to communicate with minimal copying of data. It is worth noting that other developers have attempted to solve this problem over the years, generally without success; see, for example, [the sad story of kdbus](<https://lwn.net/Articles/619068/>). Hope springs eternal, though, and perhaps io_uring is the platform upon which a successful solution can be built. 

There are facilities for direct and broadcast messages. Communication is done through "channels"; it all starts when one process issues at least one `IORING_REGISTER_IPC_CHANNEL_CREATE` operation to establish an open channel. Other processes can attach to existing channels if the permissions allow. Two basic operations, `IORING_OP_IPC_SEND` and `IORING_OP_IPC_RECV`, are used to send and receive messages, respectively. There is no documentation, naturally, but interested readers can look at [this patch](<https://lwn.net/ml/all/20260313130739.23265-3-git@danielhodges.dev>) containing a set of self-tests that exercise the new features. 

The io_uring maintainer, Jens Axboe, quickly [noticed](<https://lwn.net/ml/all/d6e64251-2025-438c-92d6-71b44927b437@kernel.dk/>) that the patch showed signs of LLM-assisted creation, something that Hodges [owned up to](<https://lwn.net/ml/all/hzb3i37w6isn7gx7jqc223fmznxxjmqvlxke2rdb3lb43htifq@j45xx427nppc/>). He also noted that the series falls short of being a complete D-Bus replacement, lacking features like credential management. Still Axboe [agreed](<https://lwn.net/ml/all/54f95b7a-45de-4353-9308-12cd64dbe894@kernel.dk/>) that an IPC feature for io_uring ""makes sense to do"" and seemed happy with the overall design of the code. Some questions he asked though, went unanswered. For this work to proceed, Hodges will need to return and do the hard work to bring a proof-of-concept patch up to the level needed for integration into a core subsystem like io_uring. 

#### Bus1 returns

Back in 2016, David ~~Herrmann~~ Rheinsberg proposed [a new kernel subsystem called "bus1"](<https://lwn.net/Articles/697191/>), which would provide kernel-mediated interprocess communication along the lines of D-Bus. It allowed the passing of messages, but also of capabilities, represented by bus1 handles and open file descriptors. The proposal attracted some attention, and brought some interesting ideas (see the above-linked article for details), but stalled fairly quickly and was never seriously considered for merging into the mainline kernel. 

Ten years later, [bus1 is back](<https://lwn.net/ml/all/20260331190308.141622-1-david@readahead.eu>)~~, posted this time by David Rheinsberg~~. The code has seen a few changes in the intervening decade: 

> The biggest change is that we stripped everything down to the basics and reimplemented the module in Rust. It is a delight not having to worry about refcount ownership and object lifetimes, but at the cost of a C<->Rust bridge that brings some challenges. 

The core features of bus1 remain similar to what was proposed in 2016. For the time being, Rheinsberg is focusing on the Rust aspects of the work and requesting help from the Rust for Linux community to get that integration into better shape. 

At some future time, presumably, the new bus1 implementation will be more widely exposed within the kernel community, at which point we will see if there is an appetite for this kind of in-kernel IPC mechanism or not. For those who would like an early look, [this patch](<https://lwn.net/ml/all/20260331190308.141622-8-david@readahead.eu>) contains documentation on how the bus1 API will work, though with a number of details left unspecified. 

[_Editor's note: we originally missed that David had changed his name. Apologies for the error._]
