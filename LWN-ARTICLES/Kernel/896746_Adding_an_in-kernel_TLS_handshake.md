---
title: "Adding an in-kernel TLS handshake"
url: https://lwn.net/Articles/896746/
date: "June 1, 2022"
category: "Networking-Protocols"
author: "By Jake Edge June 1, 2022 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 1, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Adding [support for an in-kernel TLS handshake](<https://lwn.net/Articles/892216/>) was the topic of a combined storage and filesystem session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). Chuck Lever and Hannes Reinecke led the discussion on ways to add that support; they are interested in order to provide TLS for network storage and filesystems. But there are likely other features, such as [QUIC](<https://lwn.net/Articles/558826/>) support, that could use an in-kernel TLS implementation. 

#### Problem

Reinecke started things off by saying that, while Lever was interested in the feature for NFS, he wanted it for NVMe. The problem is that those applications cannot use the current in-kernel TLS support because they need to initiate the handshake from the kernel, Reinecke said. Current kernels can communicate using TLS, but the connection handshake is done in user space, then the connected socket is passed to the kernel for sending and receiving the data. 

The reason the existing mechanism cannot be used is because there is already a socket connected to the remote host within the kernel that is, effectively, being converted to use TLS. So there is a need to pass a connected socket from the kernel to user space if the handshake will be done there, but there is no existing mechanism to do that. 

[ ![\[Chuck Lever and Hannes Reinecke\]](https://static.lwn.net/images/2022/lsfmm-lever-reinecke-sm.png) ](<https://lwn.net/Articles/896837/>)

An alternative would be to do the whole job within the kernel, as a company called [Tempesta](<https://tempesta-tech.com/>) has done, Reinecke said. That works, but it brings "a lot of security-relevant code" into the kernel, which would require an audit to help limit the potential security danger. Someone suggested writing that code in Rust; "we did think of that", Lever said with a chuckle. In any case, there are reasonable arguments that this kind of code should not be in the kernel at all, regardless of language, Reinecke said. 

James Bottomley asked about using the kernel as a man in the middle and passing the packets back and forth to user space as needed. Reinecke said that does not work with the existing libraries; if the kernel endpoint can be passed to user space, there are TLS libraries that can just handle the handshake directly. 

Steve French said that there is value in finding a way to create a guinea-pig implementation for dealing with the handshake as a starting point, even if that code never goes upstream. It would allow the creation of a reference platform that shows that TLS for NFS, NVMe, or, in his case, SMB over QUIC, is viable, then it can be reworked as needed. But there is no good example that he could find of an upcall passing the kernel socket to a user-space library. 

Reinecke agreed; there is no mechanism of that sort, which is why they have been pondering on how it should be done. One possibility is to update the netlink mechanism to allow passing file descriptors from the kernel to user space. Josef Bacik said that the Linux [network block device](<https://nbd.sourceforge.io/>) (NBD) already uses netlink that way, but Lever pointed out that user space creates the endpoint for NBD, not the kernel, so that is passing the socket in the opposite direction of what is needed here. 

David Howells said that for TLS 1.3 all of the necessary code should already be available in the kernel crypto subsystem. It should just be a matter of calling it properly. But Reinecke said that the crypto layer does have what is needed for encrypting and decrypting the data, but it does not have necessary pieces for the initial handshake. 

Bacik said that FreeBSD does the TLS handshake in user space and wondered how it did so. Lever said that it passed a file descriptor to a user agent that uses an existing library, probably OpenSSL, to do the handshake. That is generally how the security community recommends that it be handled. 

On the server side, the kernel will be accepting connections from clients that will then need to have a TLS connection initialized, Lever said, so there is really no way of getting around the need to pass connected sockets to user space. His initial implementation used a separate address family for a user agent's socket; the user agent would accept a connection from the kernel on that socket, which "materializes the connected endpoint in the user agent's file descriptor table". That socket gets passed by the agent to GnuTLS, which does the handshake and closes the accepted socket; that tells the kernel that the connected endpoint is ready to use. 

That prototype worked for NFS and NVMe. They are hoping to build infrastructure that QUIC can use, as well, since it uses the TLS 1.3 handshake protocol to establish connections. 

#### Direction

There was quite a bit of pushback from the networking developers when they discussed doing the handshake directly from within the kernel, Lever said. Reinecke asked if it made sense to continue exploring that option or if the user-space solution was the best route. Bacik said that he is normally "extremely allergic" to putting that kind of code in the kernel, but since the crypto pieces are already there, it does not "seem like it's a big deal" to do so. Bottomley pointed out that it is just the primitives that are present in the kernel, however; TLS has "a huge amount of handshaking code" that is missing from the kernel. 

Lever said that TLS 1.3 reduces the amount of code needed for handshaking by roughly half; both he and Reinecke only need support for 1.3. But Bottomley said that he had looked at the bug reports for OpenSSL, specifically regarding the 1.3 handshaking; the code size may be less, but there are still many bugs reported for it. 

Chris Mason said that the TLS-for-storage developers were faced with "two different slogs" to choose from; one is to add the TLS handshake code to the kernel and the other is to figure out how to add the mechanism so that it can be done in user space. Both will be a lot of work, but the user-space solution will likely be better long-term. As security problems arise with TLS, for example, it will be easier to address them in user space. If it were him, Mason said, he would choose the user-space route. 

Lever said that one area where they do not feel comfortable with the user-space solution is in handling a root filesystem or block device over TLS. The user agent process needs to be made special somehow so that the kernel can always rely on it being there if it needs to re-establish the TLS session—even when there is memory pressure, for example. 

Another problem that Lever sees is how the kernel knows that it can trust the process it is talking to. The kernel is making an upcall, but how can it be sure that it is talking to what it expects? It is a more general problem that he does not think has been solved for other user-space helpers. Ted Ts'o said that it is the same problem faced by firmware and module loading within the kernel; the assertion is that `/sbin/request_module` is sane and a similar assertion could be made for the TLS user agent binary. 

For a prototype and to work out any problems that may be encountered, it clearly makes sense to do the handshake in user space, Lever said. Every time he talks to a group of kernel developers, he feels like the chances of eventually moving that handling into the kernel dwindle. French suggested that, once there are consumers of the facility in the kernel, the networking developers may see that it makes sense to move that handling into the kernel. Reinecke agreed; it really is not a filesystem or storage topic, but something that the networking developers need to consider. 

There are two big advantages that TLS brings, which makes it a "great value add for storage protocols", Lever said. It allows both servers and clients to authenticate the other end of the connection using X.509 certificates. It also provides in-transit encryption in a way that can be offloaded to specialized hardware. TLS is well-established in the industry, which makes it a good basis for an encryption feature. 

The mechanism for passing the TLS information to the user agent is perhaps one of the more contentious pieces, Lever said. The prototype uses socket options for the new address family to pass the connection information. That allows the kernel to send certificate data, pre-shared keys, and other information specific to the TLS connection and handshake. It is seen as ugly by some of the reviewers of the prototype code, however. 

The session wound down soon after that. It would seem that, at least for now, the same basic approach will be taken, though there are still multiple issues that need to be resolved.
