---
title: Allowing BPF programs more access to the network
url: https://lwn.net/Articles/1022034/
date: "May 28, 2025"
category: BPF
author: "By Daroc Alden May 28, 2025 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Daroc Alden**  
May 28, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Mahé Tardy led two sessions about some of the challenges that he, Kornilios Kourtis, and John Fastabend have run into in their work on [ Tetragon](<https://tetragon.io/>) (Apache-licensed BPF-based security monitoring software) at the Linux Storage, Filesystem, Memory Management, and BPF Summit. The session prompted discussion about the feasibility of letting BPF programs send data over the network, as well as potential new kfuncs to let BPF firewalls send TCP reset packets. Tardy presented several possible ways that these could be accomplished. 

#### Sending data

Tetragon has two general jobs: enforcing security policies and collecting statistics and other information for observability. The way that the latter currently works, Tardy explained, introduces unnecessary copies. BPF programs will create records of events and place them into a ring buffer. Then Tetragon's user-space component reads the events and eventually writes them to a file, a pipe, or a network socket in order to centralize and store them. 

[ ![\[Mahé Tardy\]](https://static.lwn.net/images/2025/mahe-tardy-lsfsmmbpf-small.png) ](<https://lwn.net/Articles/1022398>)

That requires a minimum of two copies between the kernel and user space. While exploring alternatives, Tardy realized that this situation could be avoided if BPF programs were allowed to call [ `vmsplice()`](<https://www.man7.org/linux/man-pages/man2/vmsplice.2.html>). The user-space agent could give the BPF program a file descriptor, and let BPF call `vmsplice()` to forward the information. Eventually, it might be possible to remove the user-space agent altogether. 

An alternative to `vmsplice()` would be to use io_uring to perform the same operations. Tardy clarified that for his use case, he really mostly cares about being able to send data over the network. Generally, Tetragon sends two types of data: alerts and periodic reports. The periodic reports are created in a timer callback, which may cause additional complications since he isn't sure whether those are called in a sleepable context. 

Andrii Nakryiko thought that a synchronous send operation — which could block for a long time — would be a bad fit for BPF. Tardy agreed, saying that an asynchronous send operation would be fine. Nakryiko thought this was a lot of effort to avoid a small number of copies. Alexei Starovoitov pointed out that there is such a thing as a [ kernel TCP socket](<https://www.kernel.org/doc/html/latest/networking/kapi.html#c.sock_create_kern>), so this is technically possible. Also, workqueues call their tasks in a sleepable context, so the operation could be run as a workqueue item and that would work. He agreed that it seemed like a lot of effort to avoid user-space copies, though. 

Tardy explained that forwarding these reports is ""almost the last thing the agent is doing"". If it could be done in BPF, Tetragon would be close to being implemented in pure BPF. Although he didn't speak to why this would be desirable, an earlier session had raised the idea of making security software harder to tamper with by avoiding user-space components, so that may have been what he had in mind. 

Starovoitov pointed out that there is an ongoing effort to use [ netconsole](<https://www.kernel.org/doc/html/latest/networking/netconsole.html>) to send kernel log messages over TCP. So perhaps Tetragon's BPF programs could be made to print to the console, which is then sent over TCP. Daniel Borkmann asked whether netconsole could send arbitrary data; Starovoitov said that it could. Tardy suggested that they could start by prototyping something using netconsole's existing UDP-based messages. The session ended without coming to a firm conclusion, but Tardy left with a number of new directions to explore. 

#### TCP reset

Currently, it is possible for BPF firewalls to drop packets, and therefore de-facto terminate a TCP connection. It would be friendlier, Tardy said in his second session, to send a TCP reset to immediately terminate the connection. This is already what other firewalls, like netfilter, do; Tardy wants to add a kfunc to let BPF programs do the same thing. 

One possible way to add that would be to extend the [ `bpf_sock_destroy()`](<https://docs.ebpf.io/linux/kfuncs/bpf_sock_destroy/>) function that [ Aditi Ghag added](<https://lwn.net/Articles/926796/>) in 2023. That function lets BPF programs close sockets in specific circumstances: while inside an iterator and holding the socket lock. The fact that it sends a TCP reset is really a side effect of its main operation, but it is somewhat related. 

Borkmann pointed out that using `bpf_sock_destroy()` would only work if the socket existed on the machine in question; a firewall sitting between a client and a server would need a different way to send a reset. Another member of the audience suggested setting up an unroutable route, forwarding a packet from the TCP connection to that, and letting the existing networking stack handle the rest. 

There is already a kernel function that allows BPF programs to send TCP acknowledgment messages; in light of that, adding one for sending reset messages struck some people as not a big deal. Ultimately, this discussion didn't reach a conclusion either, but there was no real opposition to the idea of allowing BPF programs to cleanly terminate TCP connections.
