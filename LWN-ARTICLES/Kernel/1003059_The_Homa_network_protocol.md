---
title: The Homa network protocol
url: https://lwn.net/Articles/1003059/
date: "December 30, 2024"
category: "Networking-Protocols"
author: "By Jonathan Corbet December 30, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 30, 2024

The origins of the TCP and UDP network protocols can be traced back a full 50 years. Even though networks and their use have changed radically since those protocols were designed, they can still be found behind most networking applications. Unsurprisingly, these protocols are not optimal for all situations, so there is ongoing interest in the development of alternatives. One such is the [Homa transport protocol](<https://homa-transport.atlassian.net/wiki/spaces/HOMA/overview>), developed by John Ousterhout (of [Tcl/Tk](<https://www.tcl.tk/>) and [Raft](<https://raft.github.io/>) fame, among other accomplishments), which is aimed at data-center applications. Ousterhout is currently trying to get [a minimal Homa implementation](<https://lwn.net/ml/all/20241217000626.2958-1-ouster@cs.stanford.edu>) into the kernel. 

Most networking applications are still based on TCP, which was designed for efficient and reliable transport of streams of data across a distributed Internet. Data-center applications, instead, are often dominated by large number of small messages between many locally connected hosts. The requirements of TCP, including the establishment of connections and ordering of data, add a lot of overhead to that kind of application. The design of Homa is intended to remove that overhead while taking advantage of what current data-center networking hardware can do, with a focus on minimizing the latency between a request and its response. 

#### A quick Homa overview

At its core, Homa is designed for remote procedure call (RPC) applications; every interaction on a Homa network comes down to a request and associated reply. A client will send a request message to a server that includes a unique request ID; the server will send a reply back that quotes that ID. The only state that exists on the server is held between the receipt of the request and the receipt of the response by the client. 

Much of the key to the performance of this protocol can be found in how these messages are handled. There is no connection setup; instead, the client starts transmitting the request, with no introductory handshake, to the server. There is a limit on how many bytes of this "unscheduled" request data can be sent in this manner, which is determined by the round-trip time of the network; it should be just high enough to keep the request-transmission pipeline full until an initial response can be received from the server side. The figure of about 10,000 bytes appears in some of the Homa papers. 

The initial request packet includes the length of the full request. If the request does not fit into the size allowed for the unscheduled data, the client will wait for a "grant" response before sending any more. That grant should, if the server is responding quickly, arrive just as the initial request data has finished transmitting, allowing the client to continue sending without a pause. Grants include a maximum amount of data that can be sent, and thus function like the TCP receive window. 

This machinery is intended to get a request to the server as quickly as possible, but without the need for much, if any, buffering in the network path between the two machines. Priority queues are used to manage this traffic, with unscheduled packets normally having the highest priority. Lower priorities are used for granted traffic; the requests with the least amount of data remaining to be received are given the highest priority. 

Once the server has received the full request and processed it, a response is sent back to the client. Once again, the initial bytes are sent as unscheduled packets, with grants required for the rest if the response is large enough. In the earlier descriptions of the protocol, the server would forget everything it knew about the request immediately after sending the response. That created the possibility that requests could be resent (if the response never arrives) and executed multiple times. More recent publications include an explicit acknowledgment message indicating that a response has been received, with the sender retaining the necessary state to retransmit a reply until that acknowledgment is received. 

The details of the protocol are, of course, rather more complex than described here. There are, for example, mechanisms for clamping down on the amount of unscheduled data sent if a server is finding itself overloaded. The receiving side of a message can request retransmission if an expected packet does not arrive; unlike TCP and many other protocols, Homa puts the responsibility for detecting lost packets onto the receiving side. There is also a fair amount of thought that has gone into letting systems overcommit their resources by issuing more grants than they can immediately handle; the purpose here is to keep the pipelines full even if some senders do not transmit as quickly as expected. 

See [this paper](<https://dl.acm.org/doi/10.1145/3230543.3230564>) for a more complete (and surely more correct) description of the Homa protocol, [this page](<https://github.com/PlatformLab/HomaModule/blob/main/protocol.md>), which reflects some more recent changes, and [this 2022 article](<https://lwn.net/Articles/914030/>) for more details. 

#### Homa on Linux

The Unix socket interface was designed around streams, and is not a perfect fit for Homa, but the implementation sticks with it to the extent it can. A [`socket()`](<https://man7.org/linux/man-pages/man2/socket.2.html>) call is used to create a socket for communication with any number of other systems; the `IPPROTO_HOMA` protocol type is used. Homa can run over either IPv4 or IPv6. For server systems, a [`bind()`](<https://man7.org/linux/man-pages/man2/bind.2.html>) call can be used to set up a well-known port to receive requests; clients need not bind to a port. 

Messages are sent and received, as one might expect, with [`sendmsg()`](<https://man7.org/linux/man-pages/man2/sendmsg.2.html>) and [`recvmsg()`](<https://man7.org/linux/man-pages/man2/recv.2.html>), but there are some Homa-specific aspects that developers must be aware of. When sending a message, an application must include a pointer to this structure in the `msg_control` field of the `msghdr` structure passed to `sendmsg()`: 

```
struct homa_sendmsg_args {
    	uint64_t id;
    	uint64_t completion_cookie;
        };
```

If a request is being sent, `id` should be set to zero; the protocol implementation will then assign a unique ID to the request (and write it into `id`) before sending it to the server. For a reply message, `id` should be the ID value that arrived with the request being responded to. The `completion_cookie` value, which is only used for requests, will be passed back to the caller with the reply data when it is received. 

The receive side is a bit more complicated, because Homa requires that the buffer space for replies be registered before sending the first request on a socket. To do so, the process should allocate a range of memory, then pass it into the kernel with `SO_HOMA_RCVBUF` [`setsockopt()`](<https://man7.org/linux/man-pages/man2/getsockopt.2.html>) operation, using this structure: 

```
struct homa_rcvbuf_args {
    	void *start;
    	size_t length;
        };
```

The `start` address must be page-aligned. This memory is split into individual buffers, called "bpages", each of which is `HOMA_BPAGE_SIZE` in length; that size is 64KB in the current implementation. Each message will occupy at least one bpage; large messages will be scattered across multiple, not necessarily contiguous, bpages. 

A message is received by making a call to `recvmsg()` with a pointer to this structure passed in the `msg_control` field of `struct msghdr`: 

```
struct homa_recvmsg_args {
    	uint64_t id;
    	uint64_t completion_cookie;
    	uint32_t flags;
    	uint32_t num_bpages;
    	uint32_t bpage_offsets[HOMA_MAX_BPAGES];
        };
```

The flags field describes what the caller is willing to receive; it is a bitmask that can include either or both of `HOMA_RECVMSG_REQUEST` (to receive request messages) and `HOMA_RECVMSG_RESPONSE` (to receive responses). If `id` is zero, then `HOMA_RECVMSG_RESPONSE` will cause any response message to be returned; otherwise, only a response corresponding to the provided request ID will be returned. On return, `num_bpages` will indicate the number of bpages in the registered buffer area have been used to hold the returned message; `bpage_offsets` gives the offset of each one. 

The bpages returned by this call are owned by the application at this point, and will not be used by the kernel until they have been explicitly returned. That is done with a subsequent `recvmsg()` call, where `num_bpages` and `bpage_offsets` will indicate a set of bpages to be given back. 

This code has been ""stripped down to the bare minimum"" to be able to actually transmit requests and responses across the net; it is evidently about half of the full set of Homa patches. The intent, of course, is to ease the task of reviewing the work and getting initial support into the kernel; the rest of the work can come later. In its current form, according to the cover letter, its performance ""is not very interesting"", but that is expected to improve once the rest of the work is merged. 

See [this paper](<https://www.usenix.org/system/files/atc21-ousterhout.pdf>) for more information on the Linux implementation of Homa. 

#### Prospects

The Homa protocol originates at Stanford University, with support from a number of technology companies. Academic work often does not successfully make the transition from interesting prototype into production-quality code that can be accepted into Linux. In this case, though, Ousterhout seems determined to get the code into the mainline, and is trying to do the right things to get it there. Thus far, the four postings of the code have yielded some conversations about the protocol, but have not yet resulted in a detailed review of the code. That suggests that the initial merge of Homa is not imminent. 

It does seem likely to happen at some point, though. Then, it will be a matter of whether the operators of large data centers decide that it is worth using. Complicating that question is Ousterhout's assertion (in the above-linked paper) that, even in a kernel with less overhead than Linux, CPUs simply are not fast enough to keep up with the increases in networking speed. The real future for Homa, he suggests, may be inside the networking hardware itself. In that case, the merging into Linux would be an important proof of concept that accelerates further development of the protocol, but its use in real-world deployments might be limited. It does, in any case, show how Linux is firmly at the center of protocol development for modern networks.
