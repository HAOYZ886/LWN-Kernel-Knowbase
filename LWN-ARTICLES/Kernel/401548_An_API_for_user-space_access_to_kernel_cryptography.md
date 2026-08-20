---
title: "An API for user-space access to kernel cryptography"
url: https://lwn.net/Articles/401548/
date: "August 25, 2010"
category: Cryptography
author: "By Jake Edge August 25, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
August 25, 2010

Adding an interface for user space to be able to access the kernel `crypto` subsystem--along with any hardware acceleration available--seems like a reasonable idea at first blush. But adding a huge chunk of formerly user-space code to the kernel to implement additional cryptographic algorithms, including public key cryptosystems, is likely to be difficult to sell. Coupling that with an `ioctl()`-based API, with pointers and variable length data, raises the barrier further still. Still, there are some good arguments for providing some kind of user-space interface to the `crypto` subsystem, even if the current proposal doesn't pass muster. 

Miloslav Trmač posted an [RFC patchset](<http://lwn.net/Articles/401862/>) that implements the `/dev/crypto` user-space interface. The code is derived from [cryptodev-linux](<http://home.gna.org/cryptodev-linux/>), but the new implementation was largely developed by Nikos Mavrogiannopoulos. The patchset is rather large, mostly because of the inclusion of two user-space libraries for handling multi-precision integers ([LibTomMath](<http://libtom.org/?page=features&newsitems=5&whatfile=ltm>)) and additional cryptographic algorithms ([LibTomCrypt](<http://libtom.org/?page=features&newsitems=5&whatfile=crypt>)); some 20,000 lines of code in all. That is the current implementation, though there is mention of switching to something based on [Libgcrypt](<http://directory.fsf.org/project/libgcrypt/>), which is believed to be more scrutinized as well as more actively maintained, but is not particularly small either. 

One of the key benefits of the new API is that keys can be handled completely within the kernel, allowing user space to do whatever encryption or decryption it needs without ever exposing the key to the application. That means that application vulnerabilities would be unable to expose any keys. The keys can also be wrapped by the kernel so that the application can receive an encrypted blob that it can store persistently to be loaded back into the kernel after a reboot. 

Ted Ts'o [questioned](<http://permalink.gmane.org/gmane.linux.kernel.cryptoapi/4657>) the whole idea behind the interface, specifically whether hardware acceleration would really speed things up: 

more often than not, by the time you take into account the time to move the crypto context as well as the data into kernel space and back out, and after you take into account price/performance, most hardware crypto [accelerators] have marginal performance benefits; in fact, more often than not, it's a lose. 

He was also concerned that the key handling was redundant: ""If the goal is access to hardware-escrowed keys, don't we have the TPM [Trusted Platform Module] interface for that already?"" But Mavrogiannopoulos [noted](<http://permalink.gmane.org/gmane.linux.kernel/1025542>) that embedded systems are one target for this work, ""where the hardware version of AES might be 100 times faster than the software"". He also said that the TPM interface was not flexible enough and that one goal of the new API is that ""it can be wrapped by a PKCS #11 [Public-Key Cryptography Standard for cryptographic tokens like keys] module and used transparently by other crypto libraries (openssl/nss/gnutls)"", which the TPM interface is unable to support. 

There is already support in the kernel for [key management](<http://lwn.net/Articles/210502/>), so Kyle Moffett would like to [see that used](<http://permalink.gmane.org/gmane.linux.kernel/1025776>): ""We already have one very nice key/keyring API in the kernel (see Documentation/keys.txt) that's being used for crypto keys for NFSv4, AFS, etc. Can't you just add a bunch of cryptoapi key types to that API instead?"" Mavrogiannopoulos [thinks](<http://permalink.gmane.org/gmane.linux.kernel.cryptoapi/4668>) that because the keyring API allows exporting keys to user space--something that the `/dev/crypto` API explicitly prevents--it would be inappropriate. Keyring developer David Howells [suggests](<http://permalink.gmane.org/gmane.linux.kernel.cryptoapi/4672>) an easy way around that particular problem: ""Don't provide a read() key type operation, then"". 

But the interface itself also drew complaints. To use `/dev/crypto`, an application needs to `open()` the device, then start issuing `ioctl()` calls. Each `ioctl()` operation (which are named `NCRIO_*`) has its own structure type that gets passed as the data parameter to `ioctl()`: 

```
res = ioctl(fd, NCRIO_..., &data);
```

Many of the structures contain pointers for user data (input and output), which are declared as void pointers. That necessitates using the compat_ioctl to handle 32 vs. 64-bit pointer issues, which Arnd Bergmann [disagrees](<http://permalink.gmane.org/gmane.linux.kernel/1025807>) with: ""New drivers should be written to *avoid* compat_ioctl calls, using only very simple fixed-length data structures as ioctl commands."". He doesn't think that pointers [should be used](<http://permalink.gmane.org/gmane.linux.kernel/1026048>) in the interface at all if possible: ""Ideally, you would use ioctl to control the device while you use read and write to pass actual bits of data"". 

Beyond that, the interface also mixes in `netlink`-style variable length attributes to support things like algorithm choice, initialization vector, key type (secret, private, public), key wrapping algorithm, and many additional attributes that are algorithm-specific like key length or RSA and DSA-specific values. Each of these can be tacked on as an array of (`struct nlattr`, attribute data) pairs using the same formatting as `netlink` messages, to the end of the operation-specific structure for most, but not all, of the operations. It is, in short, a complex interface that is reasonably well-documented in the first [patch](<http://permalink.gmane.org/gmane.linux.kernel.cryptoapi/4632>) of the series. 

Bergmann and others are also concerned about the inclusion of all of the extra code, as well: 

However, the more [significant] problem is the amount of code added to a security module. 20000 lines of code that is essentially a user-level library moved into kernel space can open up so many possible holes that you end up with a less secure (and slower) setup in the end than just doing everything in user space. 

Mavrogiannopoulos [thinks](<http://permalink.gmane.org/gmane.linux.kernel.cryptoapi/4671>) that the ""benefits outweigh the risks"" of adding the extra code, likening it to the existing encryption and compression facilities in the kernel. The difference, as Bergmann points out, is that the kernel actually uses those facilities itself, so they must be in the kernel. The additional code being added here is strictly to support user space. 

In the patchset introduction, Trmač lists a number of arguments for adding more algorithms to the kernel and providing a user-space API, most of which boil down to various government specifications that require a separation between the crypto provider and user. The intent is to keep the key material separate from the--presumably more vulnerable--user-space programs, but there are other ways to do that, including have a root daemon that offers the needed functionality as noted in the introduction. There is a worry that the overhead of doing it that way would be too high: ""this would be slow due to context switches, scheduler mismatching and all the IPC overhead"". However, no numbers have yet been offered to show how much overhead is added. 

There are a number of interesting capabilities embodied in the API, in particular for handling keys. A master AES key can be set for the subsystem by a suitably privileged program which will then be used to encrypt and wrap keys before they are handed off to user space. None of the key handling is persistent across reboots, so user space will have to store any keys that get generated for it. Using the master key allows that, without giving user space access to anything other than an encrypted blob. 

All of the expected operations are available through the interface: encrypt, decrypt, sign, and verify. Each is accessible from a session that gets initiated by an NCRIO_SESSION_INIT `ioctl()`, followed by zero or more NCRIO_SESSION_UPDATE calls, and ending with a NCRIO_SESSION_FINAL. For one-shot operations, there is also a NCRIO_SESSION_ONCE call that handles all three of those operations in one call. 

While it seems to be a well thought-out interface, with room for expansion to handle unforeseen algorithms with different requirements, it's also very complex. Other than the separation of keys and faster encryption for embedded devices, it doesn't offer that much for desktop or server users, and it adds an immense amount of code and the associated maintenance burden. In its current form, it's hard to see `/dev/crypto` making its way into the mainline, but some of the ideas it implements might--particularly if they are better integrated with existing kernel facilities like the keyring.
