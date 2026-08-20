---
title: "A netlink-based user-space crypto API"
url: https://lwn.net/Articles/410763/
date: "October 20, 2010"
category: Cryptography
author: "By Jake Edge October 20, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
October 20, 2010

User-space access to the kernel cryptography subsystem has reared its head several times of late. We [looked at one proposal](<http://lwn.net/Articles/401548/>) back in August that had a `/dev/crypto` interface patterned after similar functionality in OpenBSD. There is another related effort, known as the [NCR API](<http://home.gna.org/cryptodev-linux/ncr.html>), and crypto API maintainer Herbert Xu has recently [posted an RFC](<https://lwn.net/Articles/410833/>) for yet another. But giving user space the ability to request that the kernel perform its computation-intensive crypto operations is not uncontroversial. 

As noted back in August, some kernel hackers are skeptical that there would be any performance gains by moving user-space crypto into the kernel. But there are a number of systems, especially embedded systems, with dedicated cryptographic hardware. Allowing user space to access that hardware will likely result in performance gains, in fact 50-100x performance improvements have been reported. 

Another problem with both the `/dev/crypto` and NCR APIs (collectively known as the cryptodev-linux modules) is the addition of an enormous amount of code to the kernel to support crypto algorithms beyond those that are already available. Those two modules have adapted user-space libraries for crypto and multi-precision integers and included them into the kernel. They are necessary to support some government crypto standards and certifications that require a separation between user space and crypto processing. So, the cryptodev-linux modules are trying to solve two separate (or potentially separate) problems: user-space access to crypto hardware acceleration and security standards compliance. 

When Xu first [put out an RFC](<https://lwn.net/Articles/410848/>) on his idea for the API (without any accompanying code) back in September, Christoph Hellwig had a rather [strongly worded reaction](<https://lwn.net/Articles/410850/>): 

doing crypto in kernel for userspace consumers [is] simply insane. It's computational intensive code which has no business in kernel space unless absolutely required (e.g. for kernel consumers). In addition to that adding the context switch overhead and address space transitions is god [awful] too. 

Xu more or less agrees with Hellwig, but sees his API as a way to provide access to the hardware crypto devices. Because Xu's API is based on netlink sockets (as opposed to `ioctl()`-based or a brand new API that the cryptodev-linux modules introduce), he is clearly [hoping](<https://lwn.net/Articles/410854/>) that it will provide a way forward without requiring such large changes to the kernel: 

FWIW I don't care about user-space using kernel software crypto at all. It's the security people that do. 

The purpose of the user-space API is to export the hardware crypto devices to user-space. This means PCI devices mostly, as things like aesni-intel [Intel AES instructions] can already be used without kernel help. 

Now as a side-effect if this means that we can shut the security people up about adding another interface then all the better. But I will certainly not go out of the way to add more crap to the kernel for that purpose. 

The netlink-based interface uses a new `AF_ALG` address family that gets passed to the initial `socket()` call. There is also a new `struct sockaddr_alg` that contains information about what type of algorithm (e.g. "hash" or "skcipher") is to be used as well as the specific algorithm name (e.g. "sha1" or "cbc(aes)") that is being requested. That structure is then passed in the `bind()` call on the socket. 

For things like hashing, where there is little or no additional information needed, an `accept()` is done on the socket, which yields an operation file descriptor. The data to be hashed is written to that descriptor and, when there is no more data to be hashed, the appropriate number of bytes (20 for sha1) are then read from the descriptor. 

It is a bit more complicated for ciphers. Before accepting the connection on the socket, a key needs to be established for a symmetric key cipher. That is done with a `setsockopt()` call using the new `SOL_ALG` level and `ALG_SET_KEY` option name and passing the key data and its length. But there are additional parameters that need to be set up for ciphers, and those are done using `sendmsg()`. 

A cipher will need to know which direction it is operating in (i.e. encrypting or decrypting) and may need an initialization vector. Those are specified with the `ALG_SET_OP` and `ALG_SET_IV` messages. Once the `accept()` has been done, those messages are sent to the operational descriptor and the cipher is ready for use. Data can be sent as messages or written to the operational descriptor, and the resulting data can then be read from that descriptor. 

There is an additional wrinkle for the "[authenticated encryption with associated data](<http://en.wikipedia.org/wiki/AEAD_block_cipher_modes_of_operation>)" (AEAD) block cipher mode, which can include authentication information (i.e. message authentication code or MAC) into the ciphertext stream. Because of that, AEAD requires two data streams, one containing the data itself and another with the associated authentication data (the MAC). This is handled in Xu's API by doing two `accept()` calls, the first for the operational descriptor, and the second for the associated data. If the cipher is operating in encryption mode, both descriptors will be written to, while the encrypted data is read from the operational descriptor. For decryption, the ciphertext is written to the operational descriptor, while the plaintext and authentication data are read from the two descriptors. 

There hasn't been much discussion, yet, of the actual code posting, but Xu's September posting elicited a number of complaints about performance, most from proponents of the cryptodev-linux modules. But it would seem that there is some real resistance to adding completely new APIs (as NCR does) or to adding a complicated `ioctl()`-based API (as `/dev/crypto` does). Now there are three competing solutions available, but it isn't at all clear that _any_ interface to the kernel crypto subsystem will be acceptable to the kernel community at large. We will have to wait to see how it all plays out.
