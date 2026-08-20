---
title: "Extending in-kernel TLS support"
url: https://lwn.net/Articles/892216/
date: "April 25, 2022"
category: "Networking-Protocols"
author: "By Jonathan Corbet April 25, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 25, 2022

The kernel [gained support for the TLS protocol](<https://lwn.net/Articles/666509/>) in the 4.13 release, which came out in September 2017. That support is incomplete, though, in that it does not provide the kernel with a way to initiate a TLS connection on its own. Instead, user space creates a socket and performs the TLS handshake before handing the socket to the kernel, which can then transfer data using TLS. The situation may be about to change as a result of [this patch series](<https://lwn.net/ml/linux-fsdevel/165030059051.5073.16723746870370826608.stgit@oracle-102.nfsv4.dev/>) from Chuck Lever — though user space will still need to remain in the picture. 

TLS, of course, allows for the transfer of encrypted data over the network; it is the protocol that lurks behind HTTPS links, among other things. At this point, a significant fraction of the data transferred over the net is encrypted in this fashion. Once a connection has been established, encrypting data to send to the other end is relatively straightforward, as is decrypting received data. Establishing the connection, though, is a more complex affair, involving, among other things, algorithm negotiation and the provision and verification of public keys for one or both ends. 

There are a few advantages to supporting TLS in the kernel, including a small performance boost and the ability to apply [socket filters](<https://www.kernel.org/doc/html/latest/networking/filter.html>). TLS session establishment, though, is less performance-critical and, due to its complexity, potentially a bigger source of bugs and security problems. So, when TLS support was added to the kernel, it focused on the data-transmission problem, leaving the difficulties of session setup to user space. That is how kernel TLS support has worked in the intervening years. 

This solution works, but there are times when it could be useful for the kernel to have the ability to initiate TLS sessions on it own; thus Lever's patch. That said, this patch set still does not bring the TLS handshake into the kernel, even though that is the desired goal eventually: 

> In the long run, our preference is to have a TLS handshake implementation in the kernel. However, it appears that would take a long time and there is some desire to avoid adding to the Linux kernel's "attack surface" without good reasons. So in the meantime we've created a prototype handshake implementation that calls out to user space where the actual handshake can be done by an existing library implementation of TLS. 

This design requires that a special user-space process be running in any context (specifically, any network namespace) where there may be a need for the kernel to initiate TLS connections. That process will create a socket using the new `AF_TLSH` ("TLS helper") address-family type, then listen on that socket. When the kernel needs to have a TLS session established, the `listen()` call will return with a connected TCP socket; the process can then talk with the remote peer to get the session established. If that negotiation is successful, a `setsockopt()` call with the new `SOL_TLS` option can be used to describe the newly established session. Closing the socket will then return it to the kernel. 

On the kernel side, instead, there is a new function to be called after the initial TCP connection has been made: 

```
int tls_client_hello_x509(struct socket *sock, void (*done)(void *data, int status),
    			      void *data, const char *priorities, key_serial_t peerid,
    			      key_serial_t cert);
```

This call will attempt to pass `sock` to the helper process; if that works, it will return zero; the negotiation will still be ongoing at that time. Once session setup succeeds (or fails), the `done()` callback will be called with the result of the operation; if a successful status is reported there, the kernel should be able to communicate over the socket using TLS. There is also `tls_client_hello_psk()`, which can be shared in situations where a pre-shared key exists. 

Why, one might ask, is this capability needed? One answer comes in the form of [a followup patch set](<https://lwn.net/ml/linux-fsdevel/165030062272.5246.16956092606399079004.stgit@oracle-102.nfsv4.dev/>) implementing [the remote procedure call (RPC) protocol over TLS](<https://datatracker.ietf.org/doc/draft-ietf-nfsv4-rpc-tls/>). That, in turn, can be used to implement the NFS filesystem protocol over encrypted connections. In the future, Lever said, there may also be interest in using this feature to support the SMB filesystem protocol over [QUIC](<https://lwn.net/Articles/745590/>) connections, assuming, of course, that the kernel actually gets QUIC support one of these years. 

The reaction to the TLS patches has been relatively muted, consisting solely of a set of Reviewed-by tags from Hannes Reinecke, who was also the author of one of the patches. The RPC-over-TLS patches, instead, have run into some [disagreement](<https://lwn.net/ml/linux-fsdevel/962bbdf09f6f446f26ea9b418ddfec60a23aed8d.camel@hammerspace.com/>) from Trond Myklebust, the maintainer of the kernel's NFS client. He argued that the setup work could be done entirely in user space by the `mount.nfs` utility. Lever [responded](<https://lwn.net/ml/linux-fsdevel/06AB6768-AA74-43AF-9B9A-D6580EA0AE86@oracle.com/>) that there are situations where, it is felt, the kernel needs to make the decision on whether TLS should be used. The conversation wound down without arriving at a conclusion, so chances are good that this is a topic that will come up at the [Linux Storage, Filesystem, and Memory-Management Summit](<https://events.linuxfoundation.org/lsfmm/>) in early May.
