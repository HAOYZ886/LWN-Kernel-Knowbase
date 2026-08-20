---
title: The kernel connection multiplexer
url: https://lwn.net/Articles/657999/
date: "September 21, 2015"
category: Networking
author: "By Jonathan Corbet September 21, 2015"
---

By **Jonathan Corbet**  
September 21, 2015

As the introduction to Tom Herbert's [kernel connection multiplexer (KCM) patch set](<https://lwn.net/Articles/657970/>) notes, TCP is often used for message-oriented communication protocols even though, as a streaming transport, it has no native support for message-oriented communications. KCM is an effort to make it easier to send and receive messages over TCP which adds a couple of other interesting features as well. 

#### KCM functionality

In its simplest form, a KCM socket can be thought of as a sort of wrapper attached to an ordinary TCP socket: 

> ![\[KCM diagram\]](https://static.lwn.net/images/2015/kcm1.png)

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Within the kernel, an object called a "psock" sits between the application (which is using the KCM socket) and the actual TCP connection to the world. Outgoing messages are sent as formatted by the application. Incoming data is buffered until the kernel sees (via a mechanism to be discussed momentarily) that a full message has arrived; that message is then made available to the application via the KCM socket. The psock module ensures that messages are sent and received as atomic units, freeing the application from having to manage that aspect of the protocol. 

Applications are often split into multiple processes, though, each of which can deal with incoming messages in the same way. Multiplexers to dispatch messages are thus built into many applications. But, while the kernel is handling message receipt, it can also deal with the multiplexing. So the actual architecture of KCM looks a bit more like this: 

> ![\[KCM diagram\]](https://static.lwn.net/images/2015/kcm2.png)

Each of the KCM sockets is seen as equivalent by the kernel — an incoming message can just as well be passed to any one of them. What actually happens, of course, is that the kernel chooses between one of the processes that is actually waiting for a message when one arrives; if there are no waiting processes, the message will be queued. 

Things get more complicated on the other side of the multiplexer as well, in that there could be value in having multiple TCP connections to the same destination. A phone handset, for example, might connect to a service over both broadband and WiFi. In such a situation, the multiplexer could choose between those connections for outgoing messages, and accept incoming messages on any of them. And, indeed, that's what KCM does: 

> ![\[KCM diagram\]](https://static.lwn.net/images/2015/kcm3.png)

Thus, KCM can be thought of as implementing a sort of poor hacker's [multipath TCP](<https://lwn.net/Articles/544399/>), where the application is charged with setting up the connections over the various paths. 

There is one final detail: how is the TCP data stream broken up into discrete messages? One could certainly envision building some sort of framing mechanism into KCM, as has been done in the kernel's [reliable datagram sockets](<http://linux.die.net/man/7/rds>) layer, but that would limit its flexibility when it comes to implementing _existing_ protocols. If KCM is to be usable for an existing message-oriented mechanism, there must be a way to tell KCM how the framing of messages is done. 

The solution here is perhaps predictable, since it is increasingly being used as the way to extend kernel functionality: use the [Berkeley packet filter (BPF)](<https://lwn.net/Articles/612878/>) subsystem. Whenever some data shows up at the psock level, it is passed to a BPF program for evaluation. Normally, the program will examine the data, determine the length of the message, and return that value. A return value of zero indicates that the length cannot be determined yet — more data must be received before trying again. A negative number indicates some sort of protocol error; when that happens, KCM will stop servicing the affected connection and signal an error to user space. 

#### API details

An application wanting to use KCM starts by creating a new multiplexer; this is done with a `socket()` call: 

```
#include <linux/kcm.h>
    
        kcm_socket = socket(AF_KCM, SOCK_DGRAM, KCMPROTO_CONNECTED);
```

If additional KCM sockets (attached to the same multiplexer) are needed, they can be created by calling `accept()` on the initial KCM socket. Normally, `accept()` is not applicable to datagram sockets, so this usage, while perhaps a little surprising, is arguably a reasonable way of overloading this system call. 

The application must also take care of the establishment of one or more TCP connections to the remote side and attaching them to the multiplexer. Once a socket is open, it should be placed into a `kcm_attach` structure: 

```
struct kcm_attach {
            int fd;
            int bpf_type;
            union {
                int bpf_fd;
                struct sock_fprog fprog;
            };
        };
```

Here, `fd` is the file descriptor for the open socket. The rest of this structure exists so that the application can supply the BPF program that will help split the incoming data stream into messages. If `bpf_type` is `KCM_BPF_TYPE_FD`, then `bpf_fd` contains a file descriptor corresponding to a BPF program that has been loaded with the `bpf()` system call. Otherwise, if `bpf_type` is `KCM_BPF_TYPE_PROG`, then `fprog` points to a program to be loaded directly at this time. 

The provision of two ways to load the BPF program may look a bit strange. It is, in fact, a combination of new and old ways of solving this particular problem. The `bpf()` approach is the newer way of doing things, while the `sock_fprog` approach has been used to load BPF programs into the network subsystem until now. It would not be entirely surprising to see a request from reviewers to narrow the interface down to just one of the two, most likely the `bpf()` method. 

In any case, once the structure has been filled in, it is passed to a new `ioctl()` command (`SIOCKCMATTACH`) to connect the socket to the multiplexer. There is also a `SIOCKCMUNATTACH` operation to disconnect a socket. 

Once the pieces are connected together, the application can use the KCM socket(s) like any other datagram socket. Messages can be sent and received with interfaces like `sendmsg()` and `recvmsg()` (or `write()` and `read()`), `poll()` can be used to wait for a message to arrive, and so on. The whole structure will persist until the last KCM socket is closed, at which point it is torn down. 

One might wonder why this mechanism is being proposed for the kernel, given that applications have been solving this problem in user space for years. There is no explicit justification offered in the patch set, but one can imagine that it would involve performance improvements (avoiding copying or retransmission of data), the ability to do smart load balancing, and having a single implementation used by all. The "to do" list in the patch posting, which says that ""TLS-in-kernel is a separate initiative"", suggests that there may be efforts to push other functionality into the kernel in the future. Whether these efforts will succeed remains to be seen, but there is clearly a lot of interest in adding interesting functionality to the Linux networking stack.
