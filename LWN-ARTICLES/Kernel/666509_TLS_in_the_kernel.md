---
title: TLS in the kernel
url: https://lwn.net/Articles/666509/
date: "December 2, 2015"
category: "Networking-Protocols"
author: "By Jake Edge December 2, 2015"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
December 2, 2015

An RFC [patch](<https://lwn.net/Articles/665602/>) from Dave Watson at Facebook proposes moving the bulk of [Transport Layer Security](<https://en.wikipedia.org/wiki/Transport_Layer_Security>) (TLS) processing into the kernel. There are a number of advantages he sees for doing so, but most of the commenters on the patch set seem a bit skeptical about the idea. TLS is, of course, the encryption layer that protects HTTPS and other internet protocols. 

The patch set implements [RFC 5288](<https://tools.ietf.org/html/rfc5288>) encryption for TLS, which is based on the 128-bit advanced encryption standard (AES) using Galois counter mode (GCM)—also known as "gcm(aes)". That accounts for roughly 80% of the TLS connections that Facebook sees, Watson [said](<https://lwn.net/Articles/666552/>). The idea is for the kernel to handle the symmetric encryption and decryption, while leaving the handshake processing to user space. The feature uses the user-space API to the kernel's crypto subsystem, which is accessed via sockets created using the `AF_ALG` address family. 

The basic idea is that an `AF_ALG` socket and a regular TCP socket are both created. The TCP socket is used to do the handshake with the remote endpoint, which establishes keys and such. The keys (one each for sending and receiving) are passed to the crypto socket using `setsockopt()`. An operational socket is also created by making an `accept()` call on the crypto socket. That socket is used in further processing, including setting the [initialization vectors](<https://en.wikipedia.org/wiki/Initialization_vector>) (IVs) using `sendmsg()` and control messages created using [CMSG](<http://man7.org/linux/man-pages/man3/cmsg.3.html>). There are also two IVs, one for each direction. In addition, the file descriptor for the TCP socket is passed to the operational socket in a control message; the application will then read and write data from the operational socket. Watson pointed to an [example C program](<https://github.com/djwatson/ktls>) that uses the new facility. 

That approach has a number of benefits, according to Watson. Using some additional code that was not part of his submission, he said the in-kernel TLS showed 2-7% better performance than the equivalent done in user space. The idea was inspired by some [work [PDF]](<https://people.freebsd.org/~rrs/asiabsd_2015_tls.pdf>) that Netflix did on FreeBSD to improve the performance of TLS. In addition, two other features could benefit from having TLS in the kernel, he said. The [kernel connection multiplexer](<https://lwn.net/Articles/657999/>) (KCM) needs access to unencrypted data in the kernel, which this would provide; offloading TLS encryption and decryption to NICs would also require TLS framing support in the kernel. 

But Hannes Frederic Sowa [questioned](<https://lwn.net/Articles/666570/>) two of those advantages. He believes that the existing facilities provided by Linux already do less copying than those that FreeBSD provides, so he suggested comparing the in-kernel approach with a user-space implementation using `mmap()` and `vmsplice()` on the TCP socket. Beyond that, he noted that kernel developers have been strong opponents of TCP-offloading efforts. In order to provide TLS offloading, a NIC would also need to handle the TCP layer, so it would effectively be doing TCP offloading as well. 

Crypto maintainer Herbert Xu was a bit [surprised at the approach](<https://lwn.net/Articles/666576/>). While he can see that using `AF_ALG` makes sense as a way to export TLS functionality to user space, it's not the way he might have approached it: 

However, I must say that it wouldn't have been my first pick. I'd imagine a TLS socket to look more like a TCP socket, or perhaps a KCM socket as proposed by Tom. 

But Watson noted that handling out-of-band (OOB) data is one reason to not just layer TLS on top of a TCP socket. TLS transfers data beyond just the data being sent by the application, for things like alerts or to change the cipher being used, but a TCP socket lacks an easy way to signal the reception of that kind of data. In Watson's patches, the crypto socket returns an error in that situation and user space can then read the OOB data from the TCP socket if it wishes. 

But others also questioned the value of having TLS in the kernel at all. Modern processors provide user-space programs with access to accelerated crypto instructions directly, without a need for kernel intervention. There is some crypto-acceleration hardware out there, where there might be some benefit to having TLS in the kernel, but it has mostly fallen by the wayside because of better processor support for crypto. As Sowa [put it](<https://lwn.net/Articles/666580/>): 

There are some crypto [accelerators] out there so that putting tls into the kernel would give a net benefit, because otherwise user space has to copy data into the kernel for device access and back to user space until it can finally be send out on the wire. 

Since processors provide aesni and other crypto extensions as part of their instruction set architecture, this, of course, does not make sense any more. 

Overall, it looks like it will take some more convincing arguments before putting TLS in the kernel will be seriously considered. For some specialized situations, it might make sense to do so, but even the limited version Watson posted adds more than 1200 lines of code to the kernel—for dubious gains. Over time, more and more crypto has been added to the kernel, though, so maybe TLS will eventually find its way in too.
