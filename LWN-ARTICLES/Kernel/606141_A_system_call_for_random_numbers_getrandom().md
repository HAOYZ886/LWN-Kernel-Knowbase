---
title: "A system call for random numbers: getrandom()"
url: https://lwn.net/Articles/606141/
date: "July 23, 2014"
category: "Random numbers; Security-Random number generation"
author: "By Jake Edge July 23, 2014"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
July 23, 2014

The Linux kernel already provides several ways to get random numbers, each with its own set of constraints. But those constraints may make it impossible for a process to get random data when it needs it. The [LibreSSL project](<http://www.libressl.org/>) has recently been making some noise about the lack of a "safe" way to get random data under Linux. That has led Ted Ts'o to [propose a new `getrandom()` system call](<https://lwn.net/Articles/605828/>) that would provide LibreSSL with what it needs, while also solving other kernel random number problems along the way. 

The kernel maintains random number "pools" that get fed data that comes from sampling unpredictable events (e.g. inter-key timing from the keyboard). The amount of entropy contributed by each of these events is estimated and tracked. A cryptographically secure pseudo-random number generator (PRNG) is used on the data in the pools, which then feed two separate devices: `/dev/urandom` and `/dev/random`. 

The standard way to get random numbers from the kernel is by reading from the `/dev/urandom` device. But there is also the `/dev/random` device that will block until enough entropy has been collected to satisfy the `read()` request. `/dev/urandom` should be used for essentially all random numbers required, but `/dev/random` is sometimes used for things like extremely sensitive, long-lived keys (e.g. GPG) or one-time pads. In order to use either one, though, an application has to be able to `open()` a file, which requires that there be file descriptors available. It also means that the application has to be able to see the device files, which may not be the case in some containers or `chroot()` environments. 

LibreSSL has been written to use `/dev/urandom`, but also to have a fallback if there is an exhaustion of file descriptors (which an attacker might try to arrange) or there is some other reason that the library can't open the file. The fallback is to use the deprecated [`sysctl()` system call](<http://man7.org/linux/man-pages/man2/sysctl.2.html>) to retrieve the `/proc/sys/kernel/random/uuid` value, but without actually having to open that file (since LibreSSL already knows that `/dev/urandom` could not be opened). But `sysctl()` may disappear someday—some distribution kernels have already removed it—and, sometimes, using it puts a warning into the kernel log. If the `sysctl()` fails, LibreSSL falls further back to a [scary-looking function](<https://github.com/libressl-portable/openbsd/blob/4e9048830a68da79247f30aba182b1599da139b9/src/lib/libcrypto/crypto/getentropy_linux.c#L307>) that tries to generate its own random numbers from various (hopefully) unpredictable values available to user space (e.g. timestamps, PID numbers, etc.). 

All of that can be seen in a well-commented [chunk of code](<https://github.com/libressl-portable/openbsd/blob/4e9048830a68da79247f30aba182b1599da139b9/src/lib/libcrypto/crypto/getentropy_linux.c#L97>) in LibreSSL's `getentropy_linux.c` file. The final comment in that section makes a request: 

```
* We hope this demonstrates that Linux should either retain their
    * sysctl ABI, or consider providing a new failsafe API which
    * works in a chroot or when file descriptors are exhausted.
    */
```

That new API is precisely what Ts'o has proposed. The `getrandom()` system call is well-described in his patch (now up to [version 4](<https://lwn.net/Articles/606202/>)). It is declared as follows: 

```
#include <linux/random.h>
    
        int getrandom(void *buf, size_t buflen, unsigned int flags);
```

A call will fill `buf` with up to `buflen` bytes of random data that can be used for cryptographic purposes, returning the number of bytes stored. As might be guessed, the `flags` parameter will alter the behavior of the call. In the case where `flags == 0`, `getrandom()` will block until the `/dev/urandom` pool has been initialized. If `flags` is set to `GRND_NONBLOCK`, then `getrandom()` will return -1 with an error number of `EAGAIN` if the pool is not initialized. 

The `GRND_RANDOM` flag bit can be used to switch to the `/dev/random` pool, subject to the entropy requirements of that pool. That means the call will block until the pool has the required entropy, unless the `GRND_NONBLOCK` bit is also present in `flags`, in which case it will return as many bytes as it can; it will return -1 for an error with `errno` set to `EAGAIN` if there is no entropy available at all. The call returns the number of bytes it placed into `buf` (or -1 for an error). Short reads can occur due to a lack of entropy for the `/dev/random` pool or because the call was interrupted by a signal, but reads of 256 bytes or less from `/dev/urandom` are guaranteed to return the full request once that device has been initialized. 

In the proposed man page that accompanies the patch, Ts'o shows sample code that could be used to emulate the OpenBSD `getentropy()` system call using `getrandom()`. One [complaint](<https://lwn.net/Articles/606315/>) about the patch came from Christoph Hellwig, who was concerned that Ts'o was not just implementing ""exactly the same system call"" as OpenBSD. He continued: ""Having slightly different names and semantics for the same functionality is highly annoying."" But Ts'o is trying to solve more than just the LibreSSL problem, he [said](<https://lwn.net/Articles/606316/>). `getrandom()` is meant to be a superset of OpenBSD's `getentropy()`—glibc can easily create a compatible `getentropy()`, as he showed in the patch. 

The requirement that `/dev/urandom` be initialized before `getrandom()` will return any data from that pool is one of the new features that the proposed system call delivers. Currently, there is no way for an application to know that at least 128 bits of entropy have been gathered since the system was booted (which is the requirement to properly initialize `/dev/urandom`). Now, an application can either block to wait for that to occur, or test for the condition using `GRND_NONBLOCK` and looking for `EAGAIN`. Since the behavior of `/dev/urandom` is part of the kernel ABI, it could not change, but adding this blocking to the new system call is perfectly reasonable. 

The system call also provides a way to do a non-blocking read of `/dev/random` to get a partial buffer in the event of a lack of entropy. It is a bit hard to see any real application for that—if you don't need a full buffer of high-estimated-entropy random numbers, why ask for one? In fact, the new call provides a number of ways to abuse the kernel's random number facility (requesting `INT_MAX` bytes, for example), but that isn't really any different than the existing interfaces. 

There have been lots of comments of various sorts on Ts'o's patches, but few complaints. The overall idea seems to make sense to those participating in the thread, anyway. Some changes have been made based on the comments, most notably switching to blocking by default. But the latest revision generated only comments about typos. Unless that changes, it would seem that we could see `getrandom()` in the kernel rather soon, perhaps as early as 3.17.
